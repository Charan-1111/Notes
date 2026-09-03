# TLS and HTTPS — A Beginner-Friendly Guide

## 1. Why Do We Need TLS and HTTPS?

Suppose you open a website and enter:

- Your username and password
- Credit card information
- Bank account details
- Personal messages

If this information travels over the internet as plain text, anyone between you and the server might be able to read it.

For example:

```text
Username: charan
Password: mypassword123
```

An attacker monitoring the network could capture this information.

TLS protects communication between a client and a server.

```text
Client                              Server
Browser       <--- Encrypted --->   example.com
```

HTTPS is simply HTTP communication protected using TLS.

```text
HTTPS = HTTP + TLS
```

---

# 2. What Is HTTP?

HTTP stands for **Hypertext Transfer Protocol**.

It defines how clients and servers communicate over the web.

For example, a browser may send:

```http
GET /users/123 HTTP/1.1
Host: example.com
```

The server may respond:

```http
HTTP/1.1 200 OK
Content-Type: application/json

{
  "id": 123,
  "name": "Charan"
}
```

Traditional HTTP communication is not encrypted.

```text
Client -------- Plain HTTP --------> Server
```

Anyone who can observe the connection may be able to see:

- The URL path
- Request headers
- Request body
- Response body
- Cookies
- Authentication tokens
- Submitted passwords

HTTP normally uses port `80`.

Example:

```text
http://example.com
```

---

# 3. What Is HTTPS?

HTTPS stands for **Hypertext Transfer Protocol Secure**.

It means HTTP messages are transported through a secure TLS connection.

```text
Client -------- TLS encryption --------> Server
                 HTTP inside TLS
```

HTTPS normally uses port `443`.

Example:

```text
https://example.com
```

The application still communicates using HTTP, but TLS protects the HTTP messages while they travel over the network.

```text
Application data: HTTP request
        ↓
Security layer: TLS encryption
        ↓
Transport layer: TCP
        ↓
Network layer: IP
```

For most traditional HTTPS connections, the simplified protocol stack is:

```text
HTTP
  ↓
TLS
  ↓
TCP
  ↓
IP
```

HTTP/3 is different because it uses QUIC over UDP:

```text
HTTP/3
  ↓
QUIC with TLS 1.3
  ↓
UDP
  ↓
IP
```

---

# 4. What Is TLS?

TLS stands for **Transport Layer Security**.

It is a security protocol that protects data exchanged between two systems.

TLS provides three important properties:

1. **Confidentiality**
2. **Integrity**
3. **Authentication**

Let us understand each one.

---

## 4.1 Confidentiality

Confidentiality means outsiders should not be able to understand the data.

Before encryption:

```text
password=mySecretPassword
```

After encryption, the transmitted data may look like meaningless bytes:

```text
8F A2 91 C4 73 1B 09 ...
```

Someone capturing the encrypted traffic cannot easily recover the original message without the correct encryption keys.

---

## 4.2 Integrity

Integrity means the data should not be modified secretly while traveling through the network.

Imagine that a user sends:

```text
Transfer ₹1,000 to Account A
```

An attacker should not be able to change it to:

```text
Transfer ₹1,00,000 to Account B
```

TLS uses authenticated encryption and integrity checks to detect unauthorized modifications.

If encrypted data is changed, verification fails and the corrupted message is rejected.

---

## 4.3 Authentication

Authentication means verifying the identity of the system you are communicating with.

When you open:

```text
https://www.google.com
```

your browser needs evidence that it is actually communicating with Google and not an attacker pretending to be Google.

TLS uses digital certificates for this verification.

---

# 5. TLS Versus SSL

You may hear terms such as:

- SSL certificate
- SSL connection
- SSL encryption

SSL stands for **Secure Sockets Layer**.

SSL was the older protocol. It was eventually replaced by TLS because older SSL versions had serious security weaknesses.

```text
SSL 2.0 → SSL 3.0 → TLS 1.0 → TLS 1.1 → TLS 1.2 → TLS 1.3
```

Today, secure systems normally use:

- TLS 1.2
- TLS 1.3

People still commonly say “SSL certificate,” but modern HTTPS connections actually use TLS.

---

# 6. The Main Problem TLS Solves

Imagine that Charan wants to send a secret message to a server.

Without encryption:

```text
Charan -------- "My password is 1234" --------> Server
```

An attacker on the same network may read it:

```text
Charan --------> Attacker --------> Server
                    |
                    └── Sees the password
```

With TLS:

```text
Charan -------- Encrypted message --------> Server
```

The attacker may capture the encrypted message but cannot understand its contents.

However, encryption alone is not enough.

Charan must also make sure that:

- He is talking to the correct server.
- The encryption key was exchanged safely.
- The data was not modified during transmission.

TLS handles all of these problems.

---

# 7. Important Cryptography Concepts

To understand TLS, we need to understand four basic concepts:

1. Symmetric encryption
2. Asymmetric encryption
3. Hashing
4. Digital signatures

---

# 8. Symmetric Encryption

In symmetric encryption, the same secret key is used to encrypt and decrypt data.

```text
Plain text + Secret key
          ↓
       Encryption
          ↓
    Encrypted data
```

The receiver uses the same secret key:

```text
Encrypted data + Secret key
              ↓
           Decryption
              ↓
          Plain text
```

Example:

```text
Secret key: K123

"Hello" + K123 → "X8A91P"
"X8A91P" + K123 → "Hello"
```

Common symmetric encryption algorithms include:

- AES
- ChaCha20

### Advantages

Symmetric encryption is:

- Fast
- Efficient
- Suitable for encrypting large amounts of data

### Problem

Both the client and server need the same secret key.

How can they exchange that key securely over an untrusted network?

If they simply send it:

```text
Client -------- Secret key --------> Server
```

an attacker could capture the key.

TLS solves this key-agreement problem using asymmetric cryptography and secure key-exchange algorithms.

---

# 9. Asymmetric Encryption

Asymmetric cryptography uses two related keys:

- Public key
- Private key

```text
Public key  → Can be shared
Private key → Must remain secret
```

A simplified encryption example is:

```text
Data encrypted with public key
            ↓
Can be decrypted with the corresponding private key
```

Suppose a server has:

```text
Public key:  Anyone may know it
Private key: Only the server knows it
```

The public and private keys are mathematically related, but deriving the private key from the public key should be computationally impractical.

Asymmetric cryptography is useful for:

- Digital signatures
- Authentication
- Secure key agreement

However, it is slower than symmetric encryption.

That is why TLS does not normally use asymmetric cryptography to encrypt every HTTP message.

Instead, TLS uses a hybrid approach:

```text
Asymmetric cryptography → Authentication and key establishment
Symmetric cryptography  → Actual application-data encryption
```

---

# 10. Hashing

A hash function converts input data into a fixed-length value.

Example:

```text
Input:
hello

Hash:
2cf24dba5fb0a30e...
```

A small change in the input produces a very different hash:

```text
Input:
Hello

Hash:
185f8db32271fe25...
```

A good cryptographic hash function should make it extremely difficult to:

- Recover the original input from the hash
- Find two inputs with the same hash
- Modify data without changing the hash

Common secure hash algorithms include:

- SHA-256
- SHA-384

TLS uses hash functions as part of:

- Digital signatures
- Handshake verification
- Key derivation
- Data integrity protection

Hashing is not the same as encryption.

```text
Encryption → Designed to be reversed using a key
Hashing    → Designed to be one-way
```

---

# 11. Digital Signatures

A digital signature proves that particular data was approved by the owner of a private key.

Simplified process:

```text
Data
  ↓
Hash the data
  ↓
Sign the hash using the private key
  ↓
Digital signature
```

The receiver verifies the signature using the corresponding public key.

```text
Data + Signature + Public key
              ↓
          Verification
              ↓
      Valid or Invalid
```

Digital signatures provide:

- Authentication
- Integrity
- Proof that the signer possesses the private key

In TLS, certificates and handshake messages use digital signatures to prove the server's identity.

---

# 12. What Is a TLS Certificate?

A TLS certificate is a digital document that connects a domain name to a public key.

A certificate usually contains:

- Domain name
- Public key
- Certificate owner information
- Certificate issuer
- Issue date
- Expiration date
- Allowed usages
- Digital signature from the issuer

A simplified certificate might look like:

```text
Domain: api.example.com
Public Key: ABC123...
Issuer: Let's Encrypt
Valid From: 2026-01-01
Valid Until: 2026-04-01
Signature: XYZ789...
```

The certificate tells the browser:

> A trusted certificate authority confirms that this public key belongs to this domain.

The server keeps the corresponding private key secret.

```text
Certificate contains → Public key
Server stores        → Private key
```

---

# 13. What Is a Certificate Authority?

A Certificate Authority, or CA, is an organization trusted to issue and sign certificates.

Examples include:

- Let's Encrypt
- DigiCert
- GlobalSign
- Sectigo

Operating systems and browsers contain a list of trusted root certificate authorities.

When a CA verifies that an organization controls a domain, it can issue a certificate for that domain.

```text
Trusted Root CA
      ↓ signs
Intermediate CA
      ↓ signs
Server Certificate
```

This structure is called the **chain of trust**.

---

# 14. Root, Intermediate and Server Certificates

A typical certificate chain contains three levels.

## Root certificate

The root certificate belongs to a trusted root CA.

It is generally already stored in the operating system or browser.

## Intermediate certificate

Root private keys are extremely sensitive. Therefore, root CAs usually sign intermediate CA certificates instead of directly signing every website certificate.

## Server certificate

The intermediate CA signs the certificate used by the website or API.

```text
Trusted Root Certificate
          ↓ verifies
Intermediate Certificate
          ↓ verifies
Server Certificate
          ↓ belongs to
api.example.com
```

The browser verifies the chain until it reaches a root certificate that it already trusts.

---

# 15. What Does the Browser Verify?

When a browser receives a server certificate, it checks several things.

## 15.1 Is the certificate issued by a trusted authority?

The certificate chain must lead to a trusted root CA.

## 15.2 Is the certificate currently valid?

Every certificate has a validity period.

```text
Not Before: September 1, 2026
Not After:  December 1, 2026
```

An expired or not-yet-valid certificate should not be trusted.

## 15.3 Does the domain name match?

If the user opens:

```text
https://api.example.com
```

the certificate must be valid for `api.example.com`.

A certificate for another domain should not be accepted.

## 15.4 Is the certificate allowed for server authentication?

Certificates can have restrictions on how they may be used.

## 15.5 Has the certificate been revoked?

In some cases, browsers and clients may check whether a certificate was revoked before its expiry date.

## 15.6 Can the server prove possession of the private key?

Sending a valid certificate is not sufficient. An attacker could copy someone else's public certificate.

During the TLS handshake, the real server proves that it owns the matching private key by signing handshake information.

The private key itself is never sent to the client.

---

# 16. What Happens When You Open an HTTPS Website?

Suppose you open:

```text
https://api.example.com/users
```

A simplified flow is:

```text
1. Browser resolves api.example.com using DNS
2. Browser connects to the server
3. Client and server perform the TLS handshake
4. Browser verifies the server certificate
5. Client and server establish shared session keys
6. HTTP requests and responses are encrypted
```

Let us examine the TLS handshake more closely.

---

# 17. Simplified TLS Handshake

The exact messages depend on the TLS version. The following is a conceptual explanation.

```text
Client                                      Server
  |                                            |
  | ---- Supported TLS versions/ciphers ----> |
  |                                            |
  | <---- Certificate and key information --- |
  |                                            |
  | ---- Verify certificate ----------------> |
  |                                            |
  | ---- Establish shared session keys ------ |
  |                                            |
  | <==== Encrypted HTTP communication =====> |
```

The handshake has four major goals:

1. Choose compatible security settings
2. Authenticate the server
3. Create shared encryption keys
4. Verify that the handshake was not modified

---

# 18. TLS 1.3 Handshake Example

TLS 1.3 is the modern version of TLS.

A simplified TLS 1.3 handshake looks like this:

```text
Client                                                Server
  |                                                      |
  | ClientHello                                          |
  | - Supported TLS versions                             |
  | - Supported cipher suites                            |
  | - Client key share                                   |
  | ---------------------------------------------------> |
  |                                                      |
  |                                      ServerHello     |
  |                               - Selected settings     |
  |                               - Server key share      |
  |                                      Certificate     |
  |                               CertificateVerify      |
  |                                           Finished   |
  | <--------------------------------------------------- |
  |                                                      |
  | Verify certificate and server signature              |
  | Derive shared session keys                           |
  |                                                      |
  | Finished                                             |
  | ---------------------------------------------------> |
  |                                                      |
  | <========= Encrypted application data =============> |
```

Now let us understand these messages.

---

## 18.1 ClientHello

The client begins the handshake by sending a `ClientHello`.

It may include:

- Supported TLS versions
- Supported cipher suites
- Random data
- Supported cryptographic groups
- Client key share
- Server Name Indication
- Supported application protocols

Conceptually:

```text
ClientHello:
- I support TLS 1.2 and TLS 1.3
- I support these encryption algorithms
- I want api.example.com
- Here is my temporary key-exchange information
```

---

## 18.2 ServerHello

The server chooses compatible settings.

Conceptually:

```text
ServerHello:
- We will use TLS 1.3
- We will use this cipher suite
- Here is my temporary key-exchange information
```

Using the client's and server's key-exchange information, both sides independently calculate the same shared secret.

The shared secret itself is not transmitted directly.

---

## 18.3 Certificate

The server sends its certificate chain to the client.

```text
Server:
Here is my certificate for api.example.com.
```

The client verifies:

- Domain name
- Certificate validity
- Certificate chain
- Digital signatures
- Trusted root
- Allowed certificate usage

---

## 18.4 CertificateVerify

The server signs part of the handshake using its private key.

This proves that the server actually possesses the private key corresponding to the public key in the certificate.

```text
Certificate says:
"This public key belongs to api.example.com."

CertificateVerify proves:
"I possess the matching private key."
```

---

## 18.5 Finished Messages

Both sides send a protected `Finished` message calculated from the entire handshake.

This confirms that:

- Both sides derived the correct keys
- The handshake messages were not secretly modified
- Secure communication can begin

---

# 19. How Are Session Keys Created?

Modern TLS commonly uses ephemeral Diffie–Hellman key exchange, frequently through ECDHE.

ECDHE stands for:

```text
Elliptic Curve Diffie-Hellman Ephemeral
```

The important idea is that the client and server exchange public information but never send the final shared secret.

Conceptually:

```text
Client private value + Server public value
                    ↓
              Shared secret

Server private value + Client public value
                    ↓
              Same shared secret
```

An attacker may observe the public values but should not be able to calculate the shared secret efficiently.

The final shared secret is passed through a key-derivation process to create separate keys for purposes such as:

- Client-to-server encryption
- Server-to-client encryption
- Handshake protection

After that, fast symmetric encryption protects application data.

---

# 20. Why Doesn't TLS Use Only Asymmetric Encryption?

Asymmetric cryptography is computationally more expensive than symmetric encryption.

A web application may transfer:

- HTML
- JSON
- Images
- Videos
- JavaScript
- Large files

Encrypting all of this directly using asymmetric cryptography would be inefficient.

Therefore, TLS combines both techniques:

```text
Authentication/key agreement:
Asymmetric cryptography

Actual data transfer:
Symmetric cryptography
```

It gets the important benefits of both:

```text
Asymmetric cryptography → Safe authentication and key establishment
Symmetric cryptography  → Fast encrypted communication
```

---

# 21. What Is a Cipher Suite?

A cipher suite identifies cryptographic algorithms used by a TLS connection.

A TLS 1.3 cipher suite may look like:

```text
TLS_AES_128_GCM_SHA256
```

This means:

```text
TLS       → TLS protocol
AES_128   → Symmetric encryption using a 128-bit AES key
GCM       → Authenticated encryption mode
SHA256    → Hash function used during key derivation
```

Another example is:

```text
TLS_CHACHA20_POLY1305_SHA256
```

In TLS 1.3, key exchange and authentication algorithms are negotiated separately from the cipher suite.

---

# 22. What Does TLS Encrypt?

Once the TLS connection is established, TLS encrypts HTTP information such as:

- Request path
- Query parameters
- Request headers
- Cookies
- Authorization tokens
- Request body
- Response headers
- Response body
- Status codes

For example, this request is protected:

```http
POST /payments HTTP/1.1
Host: api.example.com
Authorization: Bearer secret-token
Content-Type: application/json

{
  "amount": 1000,
  "account": "ABC123"
}
```

An observer should not be able to read these HTTP contents from the encrypted connection.

---

# 23. What Does HTTPS Not Hide?

HTTPS does not make every part of network communication invisible.

An observer may still learn information such as:

- Server IP address
- Client IP address
- Connection timing
- Approximate amount of transferred data
- Destination domain in some situations
- DNS queries when unencrypted DNS is used

HTTPS protects the HTTP contents in transit, but it does not automatically provide complete anonymity.

HTTPS also cannot protect data after it reaches an insecure application.

For example:

```text
HTTPS securely delivers password
              ↓
Server stores password in plain text
              ↓
Database leak exposes password
```

TLS protected the data during transport, but the server still handled it insecurely.

---

# 24. TLS Protects Data in Transit

TLS mainly protects **data in transit**.

```text
Client storage
      ↓
Data in transit ← TLS protects this
      ↓
Server storage
```

It does not automatically protect:

- Data stored in a database
- Data written to application logs
- Screenshots
- Compromised client devices
- Compromised servers
- Malicious browser extensions
- Vulnerable application code

Security must exist at multiple layers.

---

# 25. Man-in-the-Middle Attack

A man-in-the-middle attack happens when an attacker secretly positions themselves between a client and server.

Without proper certificate verification:

```text
Client <----> Attacker <----> Real Server
```

The attacker may try to:

- Read messages
- Modify requests
- Modify responses
- Steal authentication credentials
- Impersonate the server

TLS certificate validation helps prevent this.

If the attacker presents a fake certificate for `bank.example.com`, the browser should reject it because it is not signed by a trusted CA for that domain.

```text
Browser:
"I asked for bank.example.com,
but this certificate cannot be trusted."

Result:
Connection blocked or security warning displayed
```

This protection depends on the client correctly validating certificates. Disabling certificate verification removes an essential part of TLS security.

---

# 26. Why You Should Never Disable Certificate Verification

Developers sometimes solve certificate errors by disabling verification.

For example, insecure development code may conceptually say:

```text
Skip certificate verification: true
```

This is dangerous because encryption without authentication can still allow an attacker to impersonate the server.

```text
Encrypted connection to the attacker
is still the wrong connection.
```

In production:

- Do not disable certificate verification.
- Fix the certificate or trust configuration.
- Ensure the hostname matches.
- Provide the correct CA certificate when using a private CA.

---

# 27. Certificate Domain Matching

Certificates list the domains for which they are valid using the **Subject Alternative Name**, or SAN, extension.

Example:

```text
DNS Names:
- example.com
- www.example.com
- api.example.com
```

This certificate is valid for those names.

It is not automatically valid for:

```text
admin.example.com
another-example.com
```

A wildcard certificate may contain:

```text
*.example.com
```

It can generally match one subdomain level, such as:

```text
api.example.com
shop.example.com
```

It generally does not match:

```text
v1.api.example.com
```

A wildcard certificate also does not automatically cover the root domain `example.com` unless that domain is separately included.

---

# 28. Server Name Indication

Many websites can share the same IP address.

For example:

```text
203.0.113.10:
- api.example.com
- shop.example.com
- blog.example.com
```

The server needs to know which certificate it should present.

SNI, or **Server Name Indication**, lets the client specify the requested hostname during the TLS handshake.

```text
ClientHello:
Server name = api.example.com
```

The server can then return the correct certificate.

Traditionally, the hostname in SNI may be visible to network observers. Encrypted Client Hello, when supported and correctly configured, is designed to protect more of the initial handshake metadata.

---

# 29. TLS Termination

In production architectures, the application server may not handle TLS directly.

TLS may terminate at:

- A load balancer
- A reverse proxy
- An API gateway
- An ingress controller
- A CDN

Example:

```text
Client
  |
  | HTTPS
  v
Load Balancer
  |
  | HTTP or HTTPS
  v
Go Application
```

The load balancer:

1. Receives the HTTPS connection
2. Performs the TLS handshake
3. Decrypts the request
4. Forwards the request to the backend

This is called **TLS termination**.

---

## Should the Internal Connection Use HTTP or HTTPS?

### Option 1: HTTP internally

```text
Client -- HTTPS --> Load Balancer -- HTTP --> Application
```

Advantages:

- Simple
- Lower certificate-management overhead

Risk:

- Traffic between the load balancer and application is unencrypted

This might be acceptable only inside a sufficiently controlled and trusted network, depending on security requirements.

### Option 2: HTTPS internally

```text
Client -- HTTPS --> Load Balancer -- HTTPS --> Application
```

Advantages:

- Encryption across both network segments
- Better protection in zero-trust or sensitive environments

Disadvantages:

- More certificate management
- Additional operational complexity

For sensitive systems, TLS is commonly used for internal service-to-service communication as well.

---

# 30. End-to-End TLS Versus TLS Termination

## TLS termination

```text
Client -- HTTPS --> Proxy -- HTTP --> Backend
```

The proxy decrypts the traffic.

## TLS re-encryption

```text
Client -- HTTPS --> Proxy -- HTTPS --> Backend
```

The proxy decrypts the incoming connection and creates a new TLS connection to the backend.

## TLS passthrough

```text
Client -- TLS bytes --> Proxy -- TLS bytes --> Backend
```

The proxy does not decrypt the connection. The backend handles TLS.

Each design has different effects on:

- Security
- Performance
- Routing
- Observability
- Certificate management

---

# 31. Mutual TLS

Normal HTTPS usually authenticates only the server.

```text
Client verifies Server
```

The application authenticates the user separately using:

- Username and password
- Session cookie
- JWT
- OAuth token
- API key

With mutual TLS, or mTLS, both sides present certificates.

```text
Client verifies Server
Server verifies Client
```

The flow becomes:

```text
Client certificate <---- verification ---- Server
Server certificate ---- verification ----> Client
```

mTLS is commonly used for:

- Service-to-service communication
- Internal microservices
- Banking systems
- Enterprise networks
- Zero-trust architectures
- High-security APIs

Example:

```text
Order Service -- mTLS --> Payment Service
```

The payment service accepts the connection only if the order service presents a certificate issued by an accepted CA.

---

# 32. HTTPS Does Not Replace Application Authentication

TLS authenticates the server's identity at the transport layer.

It does not automatically identify the application user.

```text
TLS:
"Is this really api.example.com?"

JWT or session:
"Which user is making this request?"
```

A backend may use both:

```text
HTTPS protects the connection
JWT authenticates the user
RBAC checks the user's permissions
```

Example:

```text
Client
  |
  | HTTPS + JWT
  v
Backend
  |
  | Verify JWT
  | Check role
  v
Protected resource
```

---

# 33. Forward Secrecy

Modern TLS commonly uses ephemeral key exchange.

This provides **forward secrecy**.

Suppose an attacker:

1. Records encrypted traffic today
2. Steals the server's long-term private key one year later

With forward secrecy, stealing the certificate's private key should not allow the attacker to decrypt previously recorded sessions.

This works because each connection uses temporary session secrets that are not derived solely from the server's long-term private key.

```text
Long-term private key compromised
              ↓
Old ephemeral session keys still unavailable
              ↓
Previously recorded sessions remain protected
```

Protecting the private key is still extremely important because attackers may use it for impersonation and other attacks.

---

# 34. TLS Session Resumption

A full TLS handshake requires cryptographic work and network messages.

When the same client reconnects, TLS can sometimes resume a previous session.

```text
First connection:
Full handshake

Later connection:
Resume using stored session information
```

Benefits include:

- Lower latency
- Reduced CPU usage
- Faster repeated connections

TLS 1.3 can use session tickets and pre-shared keys for resumption.

---

# 35. TLS 1.3 0-RTT Data

TLS 1.3 can optionally allow a returning client to send early application data using information from a previous session.

This is called **0-RTT data**.

It can reduce latency, but it has an important risk: early data may be replayed by an attacker.

Therefore, 0-RTT should not be used carelessly for non-idempotent operations such as:

```http
POST /payments
```

A replay could potentially cause the operation to execute more than once.

Safer candidates are generally read-only or replay-safe operations, but the application and infrastructure must explicitly account for replay risk.

---

# 36. HTTP to HTTPS Redirection

A server may redirect HTTP requests to HTTPS.

For example:

```http
GET /login HTTP/1.1
Host: example.com
```

Response:

```http
HTTP/1.1 301 Moved Permanently
Location: https://example.com/login
```

The browser then makes a new HTTPS request.

However, the first HTTP request was still unencrypted. An attacker could potentially interfere with that redirect.

This is why HSTS is useful.

---

# 37. HTTP Strict Transport Security

HSTS stands for **HTTP Strict Transport Security**.

A website can send this header over HTTPS:

```http
Strict-Transport-Security: max-age=31536000; includeSubDomains
```

It tells the browser:

> For the specified time, always connect to this site using HTTPS.

After learning this policy, the browser automatically converts:

```text
http://example.com
```

into:

```text
https://example.com
```

before sending the request.

This helps prevent HTTPS downgrade attacks.

Important points:

- The HSTS header must be delivered over a valid HTTPS connection.
- `includeSubDomains` extends the policy to subdomains.
- `preload` may be used when applying to browser HSTS preload lists, but it requires careful planning.
- A long HSTS duration can be difficult to undo quickly.

---

# 38. Mixed Content

A page may be loaded over HTTPS but include a resource over HTTP.

Example:

```html
<script src="http://example.com/app.js"></script>
```

This is called **mixed content**.

It is dangerous because an attacker could modify the HTTP resource.

For example, the main HTML page may be secure, but the attacker changes the unencrypted JavaScript file.

Modern browsers often block dangerous mixed content.

All page resources should use HTTPS:

```html
<script src="https://example.com/app.js"></script>
```

---

# 39. HTTPS Request Example

A user opens:

```text
https://api.example.com/profile
```

Behind the scenes:

```text
1. DNS resolves api.example.com
2. Client connects to server port 443
3. TLS handshake begins
4. Server sends certificate
5. Client verifies the certificate
6. Shared session keys are established
7. Client encrypts the HTTP request
8. Server decrypts and processes the request
9. Server encrypts the HTTP response
10. Client decrypts and displays the result
```

The protected request may logically be:

```http
GET /profile HTTP/1.1
Host: api.example.com
Authorization: Bearer eyJhbGciOi...
```

On the network, it is transmitted as encrypted TLS records rather than readable HTTP text.

---

# 40. HTTPS in a Backend Architecture

Consider the following production system:

```text
Mobile App
    |
    | HTTPS
    v
Load Balancer
    |
    | HTTPS
    v
API Gateway
    |
    | mTLS
    v
Go Backend
    |
    | TLS
    v
PostgreSQL
```

Different layers use TLS for different purposes:

| Connection | Why TLS is useful |
|---|---|
| Mobile app → Load balancer | Protect public internet traffic |
| Load balancer → API gateway | Protect internal infrastructure traffic |
| API gateway → Go backend | Authenticate and encrypt service communication |
| Go backend → Database | Protect database credentials and queries |

---

# 41. HTTPS in a Go Server

The following is a simple Go HTTPS server:

```go
package main

import (
	"fmt"
	"log"
	"net/http"
)

func helloHandler(w http.ResponseWriter, r *http.Request) {
	fmt.Fprintln(w, "Hello over HTTPS!")
}

func main() {
	mux := http.NewServeMux()
	mux.HandleFunc("/hello", helloHandler)

	server := &http.Server{
		Addr:    ":8443",
		Handler: mux,
	}

	log.Println("HTTPS server running on https://localhost:8443")

	err := server.ListenAndServeTLS(
		"server.crt",
		"server.key",
	)
	if err != nil {
		log.Fatal(err)
	}
}
```

The server requires:

```text
server.crt → Certificate and public-key information
server.key → Private key
```

Run it:

```bash
go run main.go
```

Then open:

```text
https://localhost:8443/hello
```

For local development, browsers may warn about a self-signed certificate unless the development CA is trusted and the certificate contains the correct hostname.

In production, applications are often placed behind a reverse proxy or load balancer that manages TLS.

---

# 42. Making an HTTPS Request in Go

Go verifies HTTPS certificates by default.

```go
package main

import (
	"fmt"
	"io"
	"log"
	"net/http"
)

func main() {
	response, err := http.Get("https://example.com")
	if err != nil {
		log.Fatal(err)
	}
	defer response.Body.Close()

	body, err := io.ReadAll(response.Body)
	if err != nil {
		log.Fatal(err)
	}

	fmt.Println(string(body))
}
```

The Go HTTP client performs tasks such as:

- TLS negotiation
- Certificate-chain verification
- Hostname validation
- Response decryption

Do not disable certificate verification to hide configuration problems.

---

# 43. Configuring a Go HTTPS Server

Production servers should configure timeouts.

```go
package main

import (
	"crypto/tls"
	"log"
	"net/http"
	"time"
)

func main() {
	mux := http.NewServeMux()

	mux.HandleFunc("/health", func(w http.ResponseWriter, r *http.Request) {
		w.WriteHeader(http.StatusOK)
		w.Write([]byte("healthy"))
	})

	server := &http.Server{
		Addr:              ":443",
		Handler:           mux,
		ReadHeaderTimeout: 5 * time.Second,
		ReadTimeout:       10 * time.Second,
		WriteTimeout:      15 * time.Second,
		IdleTimeout:       60 * time.Second,
		TLSConfig: &tls.Config{
			MinVersion: tls.VersionTLS12,
		},
	}

	log.Println("HTTPS server listening on port 443")

	err := server.ListenAndServeTLS(
		"/path/to/server.crt",
		"/path/to/server.key",
	)
	if err != nil && err != http.ErrServerClosed {
		log.Fatal(err)
	}
}
```

The important setting is:

```go
MinVersion: tls.VersionTLS12
```

It prevents negotiation of older TLS versions.

The exact production TLS configuration should be based on current platform guidance, client compatibility requirements and security policy.

---

# 44. Custom CA Certificates in Go

Internal services sometimes use certificates issued by a private company CA.

The client must explicitly trust that CA.

```go
package main

import (
	"crypto/tls"
	"crypto/x509"
	"fmt"
	"io"
	"log"
	"net/http"
	"os"
)

func main() {
	caCertificate, err := os.ReadFile("company-ca.crt")
	if err != nil {
		log.Fatal(err)
	}

	caPool, err := x509.SystemCertPool()
	if err != nil {
		log.Fatal(err)
	}

	if ok := caPool.AppendCertsFromPEM(caCertificate); !ok {
		log.Fatal("could not add company CA certificate")
	}

	transport := &http.Transport{
		TLSClientConfig: &tls.Config{
			RootCAs:    caPool,
			MinVersion: tls.VersionTLS12,
		},
	}

	client := &http.Client{
		Transport: transport,
		Timeout:   10 * time.Second,
	}

	response, err := client.Get("https://internal-api.example.com")
	if err != nil {
		log.Fatal(err)
	}
	defer response.Body.Close()

	body, err := io.ReadAll(response.Body)
	if err != nil {
		log.Fatal(err)
	}

	fmt.Println(string(body))
}
```

This is the correct approach for trusting a private CA.

An unsafe approach would be:

```go
// Do not use this in production.
InsecureSkipVerify: true
```

---

# 45. Basic mTLS Server in Go

The server needs to trust the CA that issued client certificates.

```go
package main

import (
	"crypto/tls"
	"crypto/x509"
	"fmt"
	"log"
	"net/http"
	"os"
)

func main() {
	clientCABytes, err := os.ReadFile("client-ca.crt")
	if err != nil {
		log.Fatal(err)
	}

	clientCAPool := x509.NewCertPool()

	if ok := clientCAPool.AppendCertsFromPEM(clientCABytes); !ok {
		log.Fatal("could not load client CA")
	}

	mux := http.NewServeMux()

	mux.HandleFunc("/secure", func(w http.ResponseWriter, r *http.Request) {
		fmt.Fprintln(w, "Client certificate accepted")
	})

	server := &http.Server{
		Addr:    ":8443",
		Handler: mux,
		TLSConfig: &tls.Config{
			MinVersion: tls.VersionTLS12,
			ClientAuth: tls.RequireAndVerifyClientCert,
			ClientCAs:  clientCAPool,
		},
	}

	log.Fatal(
		server.ListenAndServeTLS(
			"server.crt",
			"server.key",
		),
	)
}
```

Now the server accepts only clients with a valid certificate issued by the trusted client CA.

---

# 46. Self-Signed Certificates

A self-signed certificate is signed using its own private key instead of a trusted public CA.

```text
Certificate issuer = Certificate subject
```

Self-signed certificates can be useful for:

- Local development
- Testing
- Controlled internal environments

Browsers do not trust them automatically.

A browser warning does not necessarily mean the traffic is completely unencrypted. It means the browser cannot establish trust in the certificate's identity.

For internal production systems, organizations commonly use a private CA and distribute its root certificate securely to authorized clients.

---

# 47. Certificate Renewal

Certificates expire.

If a certificate expires, clients may receive errors such as:

```text
Certificate has expired
```

Therefore, production systems should automate:

- Certificate issuance
- Certificate renewal
- Certificate installation
- Expiry monitoring
- Service reload or restart when required

Let's Encrypt certificates, for example, are commonly renewed using automation.

Certificate-expiry alerts should be configured well before the expiration date.

---

# 48. Common TLS Errors

## Certificate expired

```text
x509: certificate has expired
```

Possible causes:

- Certificate was not renewed
- Renewed certificate was not deployed
- Server is still using an old certificate
- System clock is wrong

## Hostname mismatch

```text
x509: certificate is valid for example.com,
not api.example.com
```

The requested hostname is not present in the certificate's SAN list.

## Unknown certificate authority

```text
x509: certificate signed by unknown authority
```

Possible causes:

- Self-signed certificate
- Missing root CA
- Missing intermediate certificate
- Client does not trust the private CA

## Incomplete certificate chain

The server may send its own certificate without required intermediate certificates.

Some clients may still work because they cached the intermediate certificate, while others fail.

## Protocol-version mismatch

The client and server cannot agree on a TLS version.

Example:

```text
Client supports only TLS 1.0
Server requires TLS 1.2 or later
```

## Cipher mismatch

The client and server do not support any compatible cryptographic configuration.

## Wrong system time

Certificate validation depends on time.

A device with an incorrect clock may consider a valid certificate expired or not yet valid.

---

# 49. How HTTPS Helps With Common Attacks

| Attack | Does HTTPS help? | Explanation |
|---|---:|---|
| Network eavesdropping | Yes | Traffic contents are encrypted |
| In-transit modification | Yes | Unauthorized changes are detected |
| Fake server impersonation | Yes | Certificate validation authenticates the server |
| SQL injection | No | This is an application-code vulnerability |
| Cross-site scripting | No | This is an application/content vulnerability |
| Weak passwords | No | TLS cannot improve a weak password |
| Compromised server | No | Data can be stolen after decryption |
| Malware on client | Usually no | Malware may read data before encryption |
| Database leak | No | TLS does not secure stored data automatically |

HTTPS is necessary, but it is not complete application security.

---

# 50. Performance Cost of TLS

TLS adds some overhead:

- TLS handshake network messages
- Certificate verification
- Cryptographic calculations
- Encryption and decryption

Modern TLS is highly optimized, and the performance cost is generally small compared with the security benefit.

Techniques that improve performance include:

- TLS 1.3
- Session resumption
- Persistent connections
- HTTP/2 connection reuse
- Hardware acceleration
- Load-balancer TLS termination
- Proper certificate-chain configuration

In normal production systems, avoiding HTTPS for performance reasons is rarely justified.

---

# 51. HTTPS, HTTP/1.1, HTTP/2 and HTTP/3

HTTPS can carry different HTTP versions.

## HTTP/1.1 over TLS

```text
HTTP/1.1
   ↓
TLS
   ↓
TCP
```

## HTTP/2 over TLS

```text
HTTP/2
   ↓
TLS
   ↓
TCP
```

HTTP/2 supports multiplexing multiple request and response streams over a connection.

## HTTP/3 over QUIC

```text
HTTP/3
   ↓
QUIC with integrated TLS 1.3
   ↓
UDP
```

During TLS negotiation, the client and server can use ALPN, or **Application-Layer Protocol Negotiation**, to select an application protocol such as HTTP/2.

Conceptually:

```text
Client:
"I support HTTP/2 and HTTP/1.1."

Server:
"Use HTTP/2."
```

---

# 52. HTTPS and Proxies

When a client accesses an HTTPS site through an HTTP proxy, it may use the `CONNECT` method.

Conceptually:

```http
CONNECT api.example.com:443 HTTP/1.1
Host: api.example.com:443
```

The proxy creates a tunnel.

```text
Client === TLS through tunnel ===> api.example.com
```

In a simple tunnel, the proxy transports encrypted bytes without decrypting them.

Some corporate inspection systems install their own trusted CA on managed devices and create separate TLS connections to inspect traffic. This works only because the managed client has been configured to trust that CA.

---

# 53. Secure Cookies and HTTPS

Authentication cookies should usually include the `Secure` attribute.

```http
Set-Cookie: session=abc123; Secure; HttpOnly; SameSite=Lax
```

Attributes:

- `Secure`: Send the cookie only over HTTPS
- `HttpOnly`: Prevent normal JavaScript access to the cookie
- `SameSite`: Help reduce cross-site request forgery risks

HTTPS protects cookies in transit, while cookie attributes provide additional browser-side protections.

---

# 54. Certificate Pinning

Certificate pinning means a client accepts only a specific certificate or public key instead of trusting every certificate issued by accepted public CAs.

It may be used in some controlled mobile or internal applications.

Potential benefit:

```text
Even if another trusted CA wrongly issues a certificate,
the pinned client may reject it.
```

Potential problems:

- Certificate rotation becomes harder
- Expired pins can break the application
- Backup-key planning is required
- Emergency certificate replacement becomes difficult

Pinning should be used only when its operational risks are properly understood.

---

# 55. Where Should Certificates and Private Keys Be Stored?

Private keys are sensitive secrets.

Do not:

- Commit them to Git
- Put them in a public container image
- Print them in logs
- Share them through chat
- Store them in application source code

Prefer:

- Cloud certificate-management services
- Kubernetes Secrets with appropriate controls
- Hardware Security Modules
- Secret-management systems
- Restricted filesystem permissions
- Automated certificate rotation

A stolen private key may allow an attacker to impersonate the service, depending on the surrounding certificate and infrastructure controls.

---

# 56. TLS in Kubernetes

A common Kubernetes architecture is:

```text
Internet
   |
   | HTTPS
   v
Ingress Controller
   |
   | HTTP or HTTPS
   v
Kubernetes Service
   |
   v
Application Pod
```

A Kubernetes TLS secret may reference:

```text
tls.crt
tls.key
```

A simplified Ingress configuration might look like:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: example-ingress
spec:
  tls:
    - hosts:
        - api.example.com
      secretName: example-tls
  rules:
    - host: api.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: api-service
                port:
                  number: 8080
```

The ingress controller receives HTTPS traffic and uses the certificate stored in `example-tls`.

Tools such as certificate managers can automate certificate issuance and renewal.

---

# 57. TLS at Different Architecture Layers

TLS may appear at several boundaries:

```text
User
  |
  | Public HTTPS
  v
CDN
  |
  | HTTPS
  v
Load Balancer
  |
  | HTTPS
  v
API Gateway
  |
  | mTLS
  v
Backend Service
  |
  | Database TLS
  v
Database
```

Security architects decide where TLS is required based on:

- Network trust boundaries
- Data sensitivity
- Compliance requirements
- Operational complexity
- Performance
- Threat model

A good default is to encrypt sensitive traffic across every untrusted network boundary.

---

# 58. Production HTTPS Checklist

## Protocol configuration

- Support modern TLS versions.
- Disable obsolete SSL and old TLS versions.
- Use strong cryptographic configurations.
- Prefer TLS 1.3 when supported.

## Certificate management

- Use a trusted CA for public services.
- Include all required intermediate certificates.
- Ensure the hostname matches the certificate.
- Automate renewal.
- Monitor certificate expiration.
- Protect private keys.

## Application configuration

- Redirect HTTP to HTTPS.
- Consider HSTS after confirming the entire domain setup.
- Mark session cookies as `Secure`.
- Never disable certificate verification.
- Avoid mixed content.
- Avoid putting secrets in URLs.

## Infrastructure

- Decide where TLS terminates.
- Encrypt internal traffic when required.
- Use mTLS when client or service identity must be verified.
- Restrict access to certificates and private keys.
- Test certificate rotation before relying on automation.

## Monitoring

Monitor:

- Certificate expiration
- Handshake failures
- TLS protocol usage
- Invalid certificate errors
- Sudden increases in HTTPS failures
- Unsupported-client connections

---

# 59. A Simple Real-World Analogy

Imagine sending an important document using a locked box.

## HTTP

You send the document in a transparent envelope.

```text
Anyone handling it may read the contents.
```

## HTTPS

You put the document inside a locked box.

```text
Only someone with the right key can open it.
```

## Certificate

The recipient shows an identity document proving:

```text
"I am the correct recipient."
```

## Certificate Authority

A trusted organization verifies and signs that identity document.

## TLS handshake

You:

1. Verify the recipient's identity
2. Agree on a secure locking method
3. Create temporary keys
4. Start exchanging locked boxes

This analogy is not cryptographically exact, but it captures the main purpose of TLS.

---

# 60. Complete TLS/HTTPS Flow

Here is the complete simplified process:

```text
1. User enters https://api.example.com.

2. DNS resolves the domain to an IP address.

3. Client opens a network connection to the server.

4. Client sends a TLS ClientHello containing:
   - Supported TLS versions
   - Supported cryptographic options
   - Key-exchange information
   - Requested server name

5. Server responds with:
   - Selected TLS settings
   - Its key-exchange information
   - Certificate chain
   - Proof that it possesses the private key

6. Client verifies:
   - Trusted certificate chain
   - Certificate validity dates
   - Domain-name match
   - Server signature

7. Client and server derive matching session keys.

8. Both sides verify that the handshake was not modified.

9. Client encrypts and sends the HTTP request.

10. Server decrypts and processes the request.

11. Server encrypts and sends the HTTP response.

12. Client decrypts the response.

13. All further application data on the connection remains
    encrypted and integrity-protected.
```

---

# 61. Common Misunderstandings

## “HTTPS means the website is trustworthy.”

Not necessarily.

HTTPS proves that the connection is securely made to the domain in the certificate.

A malicious website can also obtain a valid TLS certificate.

```text
HTTPS means:
Secure connection to this domain

HTTPS does not mean:
Everything this website says is honest
```

## “A certificate encrypts all the data.”

Not directly.

The certificate contains identity and public-key information. TLS uses the handshake to establish symmetric session keys, which encrypt the application data.

## “The public key must be kept secret.”

No.

The public key is designed to be shared.

The private key must remain secret.

## “HTTPS protects the database.”

No.

HTTPS protects data while it travels across the relevant connection. Database encryption, access control and secure password storage are separate concerns.

## “A self-signed certificate provides no encryption.”

A self-signed certificate can still enable encrypted transport.

The main problem is trust: clients cannot automatically verify who owns it.

## “If the browser shows a padlock, the application is completely secure.”

No.

The application can still contain:

- SQL injection
- Broken authentication
- Authorization bugs
- Cross-site scripting
- Insecure password storage
- Business-logic vulnerabilities

---

# 62. Interview-Friendly Summary

TLS is a security protocol that protects data in transit.

It provides:

```text
Confidentiality → Outsiders cannot read the data
Integrity       → Unauthorized changes are detected
Authentication  → The client can verify the server
```

HTTPS means:

```text
HTTP communication protected by TLS
```

During a modern TLS handshake:

1. The client and server agree on security settings.
2. The server sends its certificate.
3. The client verifies the certificate.
4. The server proves possession of the corresponding private key.
5. Both sides establish temporary shared session keys.
6. Symmetric encryption protects HTTP requests and responses.

TLS uses different cryptographic tools for different purposes:

```text
Certificates and digital signatures → Identity and authentication
Key exchange                        → Establish shared secrets
Symmetric encryption                → Fast data encryption
Hashes and authentication tags      → Integrity and verification
```

Normal HTTPS authenticates the server. Mutual TLS authenticates both the server and client.

---

# 63. Final Mental Model

Remember TLS using this simple sentence:

> TLS verifies who you are talking to, creates shared secret keys and protects the conversation from being read or modified.

Remember HTTPS using:

```text
HTTP defines the conversation.
TLS protects the conversation.
HTTPS is the combination of both.
```

The complete mental model is:

```text
Certificate
    ↓
Verifies the server's identity

TLS handshake
    ↓
Establishes shared session keys

Symmetric encryption
    ↓
Protects HTTP requests and responses

HTTPS
    ↓
Secure communication between client and server
```