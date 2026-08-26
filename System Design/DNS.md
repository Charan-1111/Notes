# DNS (Domain Name System)

DNS is one of the most important concepts in networking and backend engineering.

The simplest definition is:

> **DNS converts human-friendly domain names into IP addresses that computers use to communicate.**

Think of DNS as the **phonebook/contact list of the Internet**.

You remember:

```text
google.com
```

rather than:

```text
142.250.183.14
```

But computers ultimately need an IP address to establish a network connection.

So DNS performs roughly:

```text
google.com
    ↓
   DNS
    ↓
142.250.xxx.xxx
    ↓
Connect to that server
```

---

# 1. Why Do We Need DNS?

Imagine there were no DNS.

To visit websites, you would need to remember IP addresses like:

```text
142.250.183.14
104.18.32.47
151.101.1.69
```

instead of:

```text
google.com
cloudflare.com
reddit.com
```

That's obviously difficult for humans.

DNS gives us a mapping:

```text
Domain Name → IP Address
```

For example:

```text
api.myapp.com
      ↓
   DNS Lookup
      ↓
203.0.113.10
```

Your browser or backend service can then communicate with:

```text
203.0.113.10
```

---

# 2. Real-World Analogy

Imagine you want to call your friend **Rahul**.

You don't normally remember:

```text
Rahul = +91 9876543210
```

You simply open your contacts and search:

```text
Rahul
```

Your phone finds:

```text
Rahul → +91 9876543210
```

and calls that number.

DNS works similarly:

```text
Rahul                  google.com
  ↓                        ↓
Contacts                  DNS
  ↓                        ↓
Phone Number           IP Address
```

So:

> **Domain name = Person's name**

> **IP address = Phone number**

> **DNS = Contact list**

---

# 3. What Happens When You Enter a URL?

Suppose you enter:

```text
https://api.example.com/users
```

There are several parts:

```text
https:// api.example.com /users
   │           │           │
Protocol     Domain       Path
```

Before your browser can send:

```http
GET /users
```

it needs to know:

> Where is `api.example.com`?

That's where DNS comes in.

The simplified flow is:

```text
Browser
   │
   │ "What is the IP of api.example.com?"
   ▼
DNS
   │
   │ "203.0.113.20"
   ▼
Browser
   │
   │ Connect to 203.0.113.20
   ▼
Server
```

Only **after DNS resolution** can the normal network communication proceed.

A simplified HTTPS request lifecycle is:

```text
URL
 ↓
DNS Lookup
 ↓
IP Address
 ↓
TCP Connection
 ↓
TLS Handshake
 ↓
HTTP Request
 ↓
HTTP Response
```

This relationship is very important for backend engineering and system design.

---

# 4. Who Actually Answers the DNS Request?

There isn't one giant DNS server containing every domain on the Internet.

DNS is a:

> **Distributed hierarchical system**

Suppose you request:

```text
api.example.com
```

Conceptually, DNS resolution may involve:

```text
Your Computer
      ↓
DNS Resolver
      ↓
Root DNS Server
      ↓
TLD DNS Server (.com)
      ↓
Authoritative DNS Server
      ↓
IP Address
```

Let's understand each one.

---

# 5. DNS Resolver

Your computer normally doesn't perform the entire DNS search itself.

Instead, it asks a **DNS Resolver**.

The resolver might be provided by:

- Your ISP
- Your organization
- A public DNS provider

Conceptually:

```text
Laptop
   │
   │ "Where is example.com?"
   ▼
DNS Resolver
```

The resolver's job is:

> **Find the DNS answer for the client.**

If the resolver already knows the answer from its cache, it can return it immediately.

Otherwise, it may need to search the DNS hierarchy.

---

# 6. Root DNS Servers

Suppose we're looking for:

```text
api.example.com
```

The resolver can start with a Root DNS server.

The Root DNS server usually doesn't respond:

```text
api.example.com = 203.0.113.20
```

Instead, it effectively says:

```text
"You are looking for a .com domain.

Ask the .com TLD servers."
```

So:

```text
Resolver
   ↓
Root DNS
   ↓
"Ask .com"
```

---

# 7. TLD DNS Servers

TLD means:

> **Top-Level Domain**

Examples include:

```text
.com
.org
.net
.io
.in
.dev
```

For:

```text
example.com
```

the TLD is:

```text
.com
```

The resolver asks the `.com` DNS infrastructure:

```text
"Where can I find information about example.com?"
```

The TLD server responds with something like:

```text
"Ask example.com's authoritative DNS servers."
```

So:

```text
Root DNS
   ↓
.com TLD
   ↓
Authoritative DNS
```

---

# 8. Authoritative DNS Server

The **Authoritative DNS Server** contains the DNS records configured for the domain.

For example:

```text
example.com
```

might have records like:

```text
example.com      → 203.0.113.10
api.example.com  → 203.0.113.20
```

The resolver asks:

```text
"What is api.example.com?"
```

The authoritative server responds:

```text
api.example.com → 203.0.113.20
```

Now the resolver finally has the answer.

---

# 9. Complete DNS Lookup

Putting everything together:

```text
You type:

api.example.com
       │
       ▼
┌───────────────┐
│ Browser / OS  │
└───────┬───────┘
        │
        │ Where is api.example.com?
        ▼
┌────────────────┐
│ DNS Resolver   │
└───────┬────────┘
        │
        ▼
┌────────────────┐
│ Root DNS       │
└───────┬────────┘
        │
        │ Ask .com
        ▼
┌────────────────┐
│ .com TLD DNS   │
└───────┬────────┘
        │
        │ Ask example.com's DNS
        ▼
┌──────────────────────┐
│ Authoritative DNS    │
│ for example.com      │
└───────┬──────────────┘
        │
        │ 203.0.113.20
        ▼
┌────────────────┐
│ DNS Resolver   │
└───────┬────────┘
        │
        │ 203.0.113.20
        ▼
┌────────────────┐
│ Your Computer  │
└───────┬────────┘
        │
        │ Connect to 203.0.113.20
        ▼
      Server
```

The important sequence to remember is:

```text
Root
 ↓
TLD
 ↓
Authoritative
```

Think of it like:

```text
Root:
"Who handles .com?"

TLD:
"Who handles example.com?"

Authoritative:
"api.example.com is 203.0.113.20"
```

---

# 10. Does This Happen for Every Request?

Thankfully, **no**.

Imagine contacting several DNS servers every time your backend sends a request.

That would add unnecessary latency.

That's why DNS heavily uses:

> **Caching**

Suppose the resolver already knows:

```text
api.example.com → 203.0.113.20
```

The next request can simply return:

```text
203.0.113.20
```

without performing the entire lookup again.

There can be caching at multiple levels:

```text
Browser Cache
      ↓
Operating System Cache
      ↓
DNS Resolver Cache
      ↓
Actual DNS Hierarchy
```

Conceptually:

```text
Need api.example.com
        ↓
Browser knows it?
   YES → Use it
        ↓ NO
OS knows it?
   YES → Use it
        ↓ NO
Resolver knows it?
   YES → Use it
        ↓ NO
Perform DNS Lookup
```

This makes DNS much faster.

---

# 11. TTL — Time To Live

DNS records aren't cached forever.

Each DNS record normally has a:

> **TTL — Time To Live**

TTL tells DNS systems:

> **How long can this DNS result be cached?**

Suppose:

```text
api.example.com → 10.0.0.5

TTL = 300 seconds
```

That means the result can be cached for:

```text
300 seconds = 5 minutes
```

After the TTL expires, the resolver may need to obtain a fresh answer.

---

## Example

Imagine your old server is:

```text
api.example.com → 10.0.0.5
```

You migrate to:

```text
api.example.com → 10.0.0.8
```

You update DNS:

```text
api.example.com → 10.0.0.8
```

But some clients might temporarily continue using:

```text
10.0.0.5
```

because they still have the old DNS result cached.

After the TTL expires, they can receive:

```text
10.0.0.8
```

This is one reason DNS changes aren't necessarily visible everywhere immediately.

---

# 12. DNS Records

An authoritative DNS server stores:

> **DNS Records**

Different DNS record types have different purposes.

Important DNS record types include:

```text
A
AAAA
CNAME
MX
NS
TXT
```

---

# 13. A Record

An **A Record** maps a domain name to an **IPv4 address**.

Example:

```text
api.example.com
      ↓
203.0.113.20
```

DNS configuration:

```text
api.example.com   A   203.0.113.20
```

Remember:

```text
A = Domain → IPv4
```

---

# 14. AAAA Record

An **AAAA Record** maps a domain name to an **IPv6 address**.

Example:

```text
example.com
    ↓
2001:db8::1234
```

Remember:

```text
A     → IPv4

AAAA  → IPv6
```

---

# 15. CNAME Record

CNAME means:

> **Canonical Name**

Instead of mapping a domain directly to an IP address, it points to **another domain name**.

Example:

```text
www.example.com
        ↓
example.com
        ↓
203.0.113.20
```

You might configure:

```text
www.example.com CNAME example.com
```

Another common backend/cloud example is:

```text
api.example.com
       ↓
my-load-balancer.cloud-provider.com
       ↓
Actual Infrastructure
```

This is useful because the underlying infrastructure can change while your public hostname remains stable.

---

# 16. MX Record

MX stands for:

> **Mail Exchange**

It tells mail systems where emails for a domain should be delivered.

Suppose someone sends:

```text
hello@example.com
```

DNS might contain:

```text
example.com
    ↓
   MX
    ↓
mail.example.com
```

MX records are mainly related to email routing.

---

# 17. NS Record

NS means:

> **Name Server**

It tells DNS:

> **Which DNS servers are authoritative for this domain?**

For example:

```text
example.com
    ↓
   NS
    ↓
ns1.dns-provider.com
ns2.dns-provider.com
```

Those name servers contain or manage the DNS records for the domain.

---

# 18. TXT Record

TXT records store text associated with a domain.

Example:

```text
example.com TXT "some-verification-value"
```

They are commonly used for things such as:

```text
Domain ownership verification
Email security
SPF
DKIM
DMARC
Third-party service verification
```

As a backend engineer, you'll occasionally encounter TXT records while configuring infrastructure or third-party services.

---

# 19. DNS Record Cheat Sheet

| Record | Purpose |
|---|---|
| `A` | Domain → IPv4 |
| `AAAA` | Domain → IPv6 |
| `CNAME` | Domain → another domain |
| `MX` | Mail server |
| `NS` | Authoritative name server |
| `TXT` | Verification/configuration text |

For backend engineering, initially focus especially on:

```text
A
CNAME
NS
TTL
```

---

# 20. DNS in a Real Backend Architecture

Imagine you build a Go API:

```text
https://api.myapp.com
```

Your production architecture could look like:

```text
                     Internet
                        │
                        │ api.myapp.com
                        ▼
                       DNS
                        │
                        ▼
                 Load Balancer
                 203.0.113.50
                        │
              ┌─────────┼─────────┐
              ▼         ▼         ▼
           Go API     Go API     Go API
          Server 1   Server 2   Server 3
```

DNS might contain:

```text
api.myapp.com
      ↓
203.0.113.50
```

Notice something important:

DNS doesn't necessarily point directly to your application server.

It might point toward infrastructure such as a:

```text
Load Balancer
```

Then:

```text
DNS
 ↓
Load Balancer
 ↓
Backend Servers
```

The load balancer decides which backend receives the request.

---

# 21. DNS vs Load Balancer

This is a common source of confusion.

DNS answers:

> **"Where should I connect?"**

A Load Balancer answers:

> **"Which server should handle this request?"**

For example:

```text
api.example.com
       │
       │ DNS
       ▼
Load Balancer
203.0.113.50
       │
       │ Load Balancing
       │
   ┌───┴────┐
   ▼        ▼
Server A  Server B
10.0.1.10 10.0.1.11
```

So:

```text
DNS
Domain → Destination

Load Balancer
Traffic → Backend Instance
```

---

# 22. DNS Can Return Multiple IP Addresses

Suppose:

```text
example.com
```

has several servers:

```text
example.com → 10.0.0.1
example.com → 10.0.0.2
example.com → 10.0.0.3
```

DNS may return multiple IP addresses.

```text
             DNS
              │
       ┌──────┼──────┐
       ▼      ▼      ▼
    Server1 Server2 Server3
```

This can be used as part of traffic distribution.

However, DNS-based traffic distribution behaves differently from a dedicated load balancer.

---

# 23. DNS in Microservices

DNS becomes extremely important when working with:

```text
Microservices
Docker
Kubernetes
Cloud Infrastructure
```

Imagine you have:

```text
User Service

Order Service

Payment Service
```

The Order Service needs to call the Payment Service.

Hardcoding an IP like:

```text
10.20.5.17
```

is a bad idea because instances may restart and IP addresses may change.

Instead, services use names:

```text
payment-service
```

Conceptually:

```text
Order Service
      │
      │ payment-service
      ▼
Internal DNS
      │
      │ 10.20.5.17
      ▼
Payment Service
```

This is closely related to:

> **Service Discovery**

---

# 24. DNS in Kubernetes

You may encounter addresses such as:

```text
kafka.kafka.svc.cluster.local
```

This is heavily connected to Kubernetes DNS.

A Kubernetes Service can have a DNS name resembling:

```text
<service>.<namespace>.svc.cluster.local
```

For example:

```text
payment-service.production.svc.cluster.local
```

Breaking it down:

```text
payment-service
      │
      └── Service Name

production
      │
      └── Namespace

svc
      │
      └── Kubernetes Service

cluster.local
      │
      └── Cluster DNS Domain
```

Your application can often call:

```text
payment-service
```

instead of hardcoding pod IP addresses.

---

# 25. Why DNS Is Important in Kubernetes

Pods are dynamic.

Imagine:

```text
Payment Pod

10.1.2.5
```

The pod crashes.

```text
10.1.2.5 ❌
```

Kubernetes creates another pod:

```text
10.1.4.8
```

If your Order Service had hardcoded:

```text
10.1.2.5
```

it would stop working.

Instead, it communicates using:

```text
payment-service
```

Conceptually:

```text
Order Service
      │
      │ payment-service
      ▼
Kubernetes DNS
      │
      ▼
Payment Service
      │
      ▼
Current Backend Pods
```

The application doesn't need to care about individual pod IP changes.

---

# 26. DNS Failures From a Backend Perspective

Suppose your Go service calls:

```text
https://payment.example.com
```

and you receive:

```text
dial tcp: lookup payment.example.com: no such host
```

Your first conclusion shouldn't necessarily be:

```text
Payment server is down.
```

The problem could instead be:

```text
Application
    ↓
DNS Lookup
    ↓
FAILED
```

The application never even obtained the IP address required to attempt the connection.

---

## Another Example

Suppose you see:

```text
dial tcp: lookup kafka.kafka.svc.cluster.local: i/o timeout
```

The flow could be failing here:

```text
Go Application
      │
      │ kafka.kafka.svc.cluster.local
      ▼
DNS Lookup
      │
      X
    Timeout
```

So the problem may involve:

```text
DNS
Network connectivity
Kubernetes DNS
Service configuration
Network policies
```

rather than Kafka itself.

Understanding DNS is therefore extremely useful when debugging backend systems.

---

# 27. Useful DNS Commands

On Linux/macOS, common commands include:

## nslookup

```bash
nslookup google.com
```

This helps you see which IP addresses are associated with a hostname.

---

## dig

Another extremely useful command is:

```bash
dig google.com
```

For example:

```bash
dig api.example.com
```

might return an answer conceptually like:

```text
api.example.com. 300 IN A 203.0.113.20
```

Breaking that down:

```text
api.example.com
       │
       ├── TTL = 300 seconds
       │
       ├── Record Type = A
       │
       └── IP = 203.0.113.20
```

`dig` is particularly useful when debugging backend infrastructure.

---

# 28. DNS Doesn't Handle HTTP

This distinction is very important.

DNS doesn't care about:

```http
GET /users

POST /orders

Authorization: Bearer ...
```

DNS happens **before HTTP communication**.

For:

```text
https://api.example.com/users
```

DNS mainly cares about:

```text
api.example.com
```

It resolves the hostname.

Then other networking protocols take over:

```text
DNS
 ↓
IP Address
 ↓
TCP Connection
 ↓
TLS
 ↓
HTTP
 ↓
GET /users
```

So DNS is **not your API router**.

---

# 29. Complete Request Example

Suppose the user enters:

```text
https://api.example.com/users
```

The complete simplified lifecycle is:

```text
User
 │
 │ https://api.example.com/users
 ▼
Browser
 │
 │ Need IP for api.example.com
 ▼
DNS
 │
 │ 203.0.113.50
 ▼
Browser
 │
 │ Connect to 203.0.113.50
 ▼
TCP Connection
 │
 ▼
TLS Handshake
 │
 ▼
HTTP Request
 │
 │ GET /users
 ▼
Load Balancer
 │
 ▼
Backend Server
 │
 ▼
Application
 │
 ▼
Database
 │
 ▼
Application
 │
 ▼
HTTP Response
 │
 ▼
Browser
```

DNS is therefore one of the **first steps** in the request lifecycle.

---

# 30. Mental Model

Don't try to memorize every DNS detail initially.

Remember this picture:

```text
You type:

https://api.example.com/users

            │
            ▼

        DNS Lookup

api.example.com
       ↓
203.0.113.20

            │
            ▼

      Connect to IP

            │
            ▼

        TCP + TLS

            │
            ▼

      HTTP Request

GET /users

            │
            ▼

         Server
```

If the DNS answer isn't cached, conceptually:

```text
                 DNS Resolver
                      │
                      ▼
                    Root
                      │
                  "Ask .com"
                      ▼
                  .com TLD
                      │
            "Ask example.com's DNS"
                      ▼
               Authoritative DNS
                      │
             "203.0.113.20"
                      ▼
                  Resolver
                      │
                      ▼
                   Client
```

---

# 31. DNS in One Sentence

> **DNS is a distributed naming system that converts human-readable domain names into addresses that computers can use to communicate.**

For example:

```text
api.example.com
       ↓
      DNS
       ↓
203.0.113.20
       ↓
TCP Connection
       ↓
TLS
       ↓
HTTP Request
       ↓
Backend Server
```

---

# 32. What Should a Backend Engineer Remember?

You don't need to memorize every internal detail of DNS.

Make sure you understand these concepts well:

1. **Domain Name → IP Address**
2. **DNS Resolver**
3. **Root DNS Server**
4. **TLD DNS Server**
5. **Authoritative DNS Server**
6. **DNS Caching**
7. **TTL**
8. **A Record**
9. **AAAA Record**
10. **CNAME Record**
11. **NS Record**
12. **DNS vs Load Balancer**
13. **DNS-based Service Discovery**
14. **Kubernetes DNS**
15. **How DNS failures appear in backend applications**

The most important mental flow is:

```text
Domain Name
    ↓
DNS Resolution
    ↓
IP Address
    ↓
Connect to Server
    ↓
Send Request
```

And for DNS resolution itself:

```text
Client
  ↓
Resolver
  ↓
Root
  ↓
TLD
  ↓
Authoritative DNS
  ↓
IP Address
  ↓
Client
```

Once these two flows are clear, you have a strong foundation for understanding DNS from a backend engineer's perspective.
