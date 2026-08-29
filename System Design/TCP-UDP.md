# TCP and UDP Explained in Detail

TCP and UDP are the two most commonly used **transport-layer protocols**.

They allow applications running on different computers to communicate over a network.

A simple mental model is:

- **IP address** identifies the destination computer.
- **Port number** identifies the application on that computer.
- **TCP or UDP** defines how the data is transported.

For example:

```text
Source:       192.168.1.10:51000
Destination:  142.250.77.14:443
Protocol:     TCP
```

Here:

- `192.168.1.10` is the source computer.
- `51000` is the source application’s temporary port.
- `142.250.77.14` is the destination server.
- `443` identifies the HTTPS service.
- TCP defines how the data is delivered.

---

# 1. Where TCP and UDP Fit

A simplified networking stack looks like this:

```text
Application Layer    HTTP, DNS, SSH, SMTP
Transport Layer      TCP, UDP
Internet Layer       IP
Link Layer           Ethernet, Wi-Fi
```

Suppose an application sends `"Hello"`.

The data travels through the layers:

```text
Application data
       ↓
TCP segment or UDP datagram
       ↓
IP packet
       ↓
Ethernet or Wi-Fi frame
       ↓
Physical network
```

At the destination, the layers are processed in reverse.

---

# 2. What Is TCP?

TCP stands for:

> Transmission Control Protocol

TCP provides a **reliable, ordered, connection-oriented byte stream** between two applications.

You can think of TCP like a phone call:

1. Establish the call.
2. Confirm that both people are ready.
3. Exchange information.
4. Repeat anything that was not heard.
5. End the call properly.

TCP is commonly used when losing or reordering data would cause problems.

Examples include:

- HTTP/1.1
- HTTP/2
- HTTPS, except HTTP/3
- SSH
- Email
- File transfers
- Database connections
- Backend API communication

---

# 3. TCP Is Connection-Oriented

Before TCP sends application data, the client and server establish a connection.

This normally happens through the **three-way handshake**.

```text
Client                              Server
  |                                   |
  | -------- SYN -------------------> |
  | <----- SYN + ACK ---------------- |
  | -------- ACK -------------------> |
  |                                   |
  |       Connection established      |
```

## Step 1: SYN

The client sends a SYN packet.

It is essentially saying:

> I want to establish a connection. My initial sequence number is X.

## Step 2: SYN-ACK

The server responds with SYN-ACK.

It is saying:

> I received your request. My initial sequence number is Y.

## Step 3: ACK

The client acknowledges the server’s response.

It is saying:

> I received your sequence number. We can communicate now.

The connection is now established.

The handshake introduces some latency, but it establishes the state required for reliable communication.

---

# 4. TCP Is a Byte-Stream Protocol

TCP treats application data as a continuous sequence of bytes.

It does **not** preserve application message boundaries.

Suppose the sender performs two writes:

```text
Write("Hello")
Write("World")
```

The receiver might read:

```text
"HelloWorld"
```

Or:

```text
"Hel"
"loWor"
"ld"
```

TCP guarantees that the bytes are delivered in order:

```text
HelloWorld
```

It does not guarantee that one `write()` will produce one corresponding `read()`.

Therefore, TCP applications need a way to identify where messages begin and end.

Common framing techniques include:

- Fixed-size messages
- Delimiters such as newline characters
- Length-prefixed messages
- Structured protocols such as HTTP

A length-prefixed message might look like:

```text
5|Hello
```

Here, `5` tells the receiver that the message contains five bytes.

---

# 5. How TCP Provides Reliability

TCP provides reliability through several mechanisms.

## 5.1 Sequence Numbers

TCP assigns sequence numbers to transmitted bytes.

For example:

```text
Segment 1: bytes 1–1000
Segment 2: bytes 1001–2000
Segment 3: bytes 2001–3000
```

Sequence numbers help the receiver:

- Put data in the correct order
- Identify missing data
- Detect duplicate data

---

## 5.2 Acknowledgements

The receiver sends acknowledgements, commonly called ACKs.

```text
Sender                              Receiver
  | ---- bytes 1–1000 ------------> |
  | <--- ACK 1001 ----------------- |
```

`ACK 1001` means:

> I have received everything up to byte 1000. Send byte 1001 next.

TCP acknowledgements are generally cumulative.

One acknowledgement can confirm that multiple earlier segments were received.

---

## 5.3 Retransmission

If data is lost, TCP retransmits it.

```text
Sender                              Receiver
  | ---- Segment 1 ---------------> |
  | ---- Segment 2 ---- X            |  Lost
  | ---- Segment 3 ---------------> |
  | <--- Missing Segment 2 -------- |
  | ---- Segment 2 ---------------> |
```

TCP can detect missing data using mechanisms such as:

- Retransmission timeouts
- Duplicate acknowledgements
- Fast retransmit

The application normally does not need to implement retransmission itself.

---

## 5.4 Ordered Delivery

Packets can take different routes and arrive out of order.

For example:

```text
Sent order:      1, 2, 3
Arrival order:   1, 3, 2
Application sees: 1, 2, 3
```

TCP holds later data temporarily until the missing earlier data arrives.

This provides ordered delivery, but it can cause **head-of-line blocking**.

If Segment 2 is missing, Segment 3 may have to wait even though it has already arrived.

---

## 5.5 Duplicate Detection

Sometimes an acknowledgement is lost.

The sender may assume that the original data was lost and retransmit it.

The receiver may then receive the same data twice.

TCP uses sequence numbers to detect duplicates and prevents duplicate bytes from being delivered to the application.

---

## 5.6 Error Detection

TCP includes a checksum.

The checksum helps detect whether data was corrupted during transmission.

If corruption is detected:

1. The segment is discarded.
2. It is treated as missing.
3. TCP eventually retransmits it.

The checksum detects corruption, but it does not directly repair the corrupted segment.

---

# 6. TCP Flow Control

Flow control prevents a fast sender from overwhelming a slow receiver.

Imagine:

```text
Sender capacity:   100 MB/s
Receiver capacity: 10 MB/s
```

If the sender continuously transmits at 100 MB/s, the receiver’s buffer may become full.

To prevent this, the receiver advertises the amount of available buffer space.

This value is known as the **receive window**.

```text
Receiver: “I currently have room for 64 KB.”

Sender: “I will send only the permitted amount before waiting.”
```

As the receiving application processes data, more buffer space becomes available.

Flow control protects the **receiver**.

---

# 7. TCP Congestion Control

Congestion control prevents a sender from overwhelming the network.

Flow control and congestion control solve different problems:

| Mechanism | Protects |
|---|---|
| Flow control | The receiving application and its buffers |
| Congestion control | The network between sender and receiver |

TCP estimates how much traffic the network can handle.

A simplified process looks like this:

```text
Start with a small sending rate
              ↓
Gradually increase the rate
              ↓
Detect packet loss or congestion
              ↓
Reduce the sending rate
              ↓
Gradually increase again
```

Congestion may be detected through:

- Packet loss
- Retransmission timeouts
- Duplicate acknowledgements
- Increased network delay

Common TCP congestion-control concepts include:

- Slow start
- Congestion window
- Congestion avoidance
- Fast retransmit
- Fast recovery

Operating systems may use algorithms such as:

- CUBIC
- BBR
- Reno

---

# 8. Closing a TCP Connection

TCP communication is full-duplex.

This means both sides can send data independently.

Because of this, each direction is normally closed separately.

```text
Client                              Server
  | -------- FIN ------------------> |
  | <------- ACK ------------------- |
  | <------- FIN ------------------- |
  | -------- ACK ------------------> |
```

A `FIN` means:

> I have finished sending data.

TCP can also terminate a connection abruptly using an `RST` packet.

An RST usually means that:

- The connection was rejected
- The connection does not exist
- The application terminated unexpectedly
- An invalid packet was received
- The connection was forcibly closed

---

# 9. Important TCP Properties

TCP provides:

- Connection-oriented communication
- Reliable delivery
- Ordered delivery
- Duplicate detection
- Error detection
- Retransmission
- Flow control
- Congestion control
- Full-duplex communication

However, these features introduce costs:

- Connection-establishment latency
- Additional packet headers
- Acknowledgement overhead
- Per-connection state on the server
- Retransmission delays
- Head-of-line blocking

---

# 10. What Is UDP?

UDP stands for:

> User Datagram Protocol

UDP sends independent messages called **datagrams** without establishing a transport-level connection first.

You can think of UDP like sending postcards:

1. Write a message.
2. Put the destination address on it.
3. Send it immediately.
4. Do not automatically know whether it arrived.
5. Another postcard may arrive before it.
6. A postcard could theoretically arrive more than once.

UDP is useful when low overhead, low latency or application-controlled delivery is more important than built-in reliability.

Common uses include:

- DNS
- Video calls
- Voice calls
- Online multiplayer games
- Live streaming
- Telemetry
- Metrics
- Service discovery
- DHCP
- QUIC
- HTTP/3

---

# 11. UDP Does Not Require a Handshake

UDP does not establish a connection before sending data.

```text
Client                              Server
  | -------- Datagram ------------> |
```

The client can send a datagram immediately.

This reduces connection-establishment overhead.

However, UDP itself does not confirm that:

- The destination exists
- The server is running
- The server is listening
- The datagram arrived
- The application processed the datagram

If an application needs confirmation, it must implement acknowledgements itself.

---

# 12. UDP Is Message-Oriented

Unlike TCP, UDP preserves datagram boundaries.

Suppose the sender sends:

```text
Datagram 1: "Hello"
Datagram 2: "World"
```

The receiver receives two separate datagrams:

```text
"Hello"
"World"
```

UDP will not combine them into one datagram such as:

```text
"HelloWorld"
```

However, either datagram could be:

- Lost
- Delayed
- Duplicated
- Received out of order

If the receiving buffer is too small, the datagram may be truncated or rejected depending on the operating system and API.

The remaining part will not arrive through another UDP read.

---

# 13. What UDP Guarantees

UDP provides:

- Source and destination ports
- Datagram boundaries
- Low protocol overhead
- Basic checksum-based error detection

UDP does not inherently provide:

- Connection establishment
- Guaranteed delivery
- Retransmission
- Ordered delivery
- Duplicate protection
- Flow control
- Congestion control

This does not mean that applications using UDP cannot be reliable.

It means reliability must be implemented by the application or by another protocol built on top of UDP.

For example, QUIC runs over UDP but implements:

- Reliable delivery
- Encryption
- Congestion control
- Loss recovery
- Stream management
- Connection management

HTTP/3 runs over QUIC.

---

# 14. UDP Packet-Loss Example

Suppose a live video application sends:

```text
Frame 1
Frame 2
Frame 3
Frame 4
```

If Frame 2 is lost, retransmitting it several seconds later may not be useful.

The conversation has already moved forward.

The application may prefer this behavior:

```text
Display Frame 1
Skip missing Frame 2
Display Frame 3
Display Frame 4
```

For real-time applications, recent information can be more valuable than complete but delayed information.

This is why UDP is often used for real-time communication.

However, the application may still need to handle:

- Packet loss
- Jitter
- Congestion
- Security
- Duplicate packets
- Selective retransmission
- Adaptive quality

---

# 15. TCP vs UDP

| Feature | TCP | UDP |
|---|---|---|
| Full form | Transmission Control Protocol | User Datagram Protocol |
| Communication model | Byte stream | Datagrams |
| Connection establishment | Required | Not required |
| Reliable delivery | Yes | No built-in guarantee |
| Ordered delivery | Yes | No |
| Retransmission | Yes | No |
| Message boundaries | Not preserved | Preserved |
| Duplicate handling | Built in | Application responsibility |
| Flow control | Yes | No |
| Congestion control | Yes | No |
| Header size | Usually at least 20 bytes | 8 bytes |
| Per-connection state | Yes | Minimal transport state |
| Broadcast and multicast | Not supported directly | Supported where the network allows |
| Common uses | APIs, databases, SSH, files | Calls, games, DNS, streaming, QUIC |

---

# 16. Is UDP Always Faster Than TCP?

UDP has less built-in overhead than TCP, but it is not automatically faster for every application.

Suppose an application requires:

- Reliable delivery
- Correct ordering
- Retransmission
- Congestion control
- Duplicate detection

If UDP is used, the application must implement these features itself.

A poorly designed UDP protocol may be:

- Slower than TCP
- Less reliable
- Unfair to other network users
- Vulnerable to congestion
- Difficult to secure

A more accurate statement is:

> UDP provides fewer transport-layer features, giving the application more control over delivery and latency.

Protocol selection should be based on application requirements, not simply on the belief that UDP is always faster.

---

# 17. Real-World Examples

## File Download

Suppose you download a 1 GB file.

Every byte must arrive correctly.

If even a small portion is missing, the file could be corrupted.

TCP is appropriate because it provides:

- Reliable delivery
- Correct ordering
- Retransmission
- Flow control
- Congestion control

---

## Video Call

During a live call, receiving an audio packet five seconds late is normally useless.

The conversation has already moved forward.

The application may prefer to:

- Accept occasional packet loss
- Avoid long retransmission delays
- Prioritize recent audio and video
- Dynamically adjust quality

UDP is often used for this type of real-time communication.

---

## DNS Query

A traditional DNS request is usually small.

For example:

```text
“What is the IP address of example.com?”
```

The client sends a UDP query and waits briefly.

If no response arrives, it can retry.

DNS can also use TCP when:

- The response is large
- Reliability is required
- A zone transfer is performed
- The UDP response is truncated

Modern encrypted DNS protocols may also use HTTPS, TLS or QUIC.

---

## Database Connection

Database communication requires reliable and ordered delivery.

You do not want a database command such as:

```sql
UPDATE accounts
SET balance = balance - 1000
WHERE id = 10;
```

to arrive partially or in the wrong byte order.

Traditional databases therefore normally use TCP connections.

Examples include:

- PostgreSQL
- MySQL
- MongoDB
- Redis

---

# 18. Ports and Sockets

A server listens on a port.

Common examples include:

| Service | Common port | Transport |
|---|---:|---|
| HTTP | 80 | TCP |
| HTTPS | 443 | TCP; HTTP/3 uses UDP |
| DNS | 53 | UDP and TCP |
| SSH | 22 | TCP |
| PostgreSQL | 5432 | TCP |
| MySQL | 3306 | TCP |
| Redis | 6379 | TCP |

A TCP connection is typically identified using five values:

```text
Source IP
Source port
Destination IP
Destination port
Protocol
```

This is commonly called a **five-tuple**.

For example:

```text
10.0.0.1:51001 → 10.0.0.10:443 using TCP
10.0.0.2:51002 → 10.0.0.10:443 using TCP
10.0.0.3:51003 → 10.0.0.10:443 using TCP
```

All clients connect to the same server port, but each connection has a unique endpoint combination.

---

# 19. TCP and UDP in Go

## TCP Server

```go
package main

import (
	"bufio"
	"fmt"
	"net"
)

func main() {
	listener, err := net.Listen("tcp", ":8080")
	if err != nil {
		panic(err)
	}
	defer listener.Close()

	fmt.Println("TCP server listening on :8080")

	for {
		conn, err := listener.Accept()
		if err != nil {
			fmt.Println("accept error:", err)
			continue
		}

		go handleConnection(conn)
	}
}

func handleConnection(conn net.Conn) {
	defer conn.Close()

	reader := bufio.NewReader(conn)

	message, err := reader.ReadString('\n')
	if err != nil {
		fmt.Println("read error:", err)
		return
	}

	fmt.Println("Received:", message)

	_, err = conn.Write([]byte("Message received\n"))
	if err != nil {
		fmt.Println("write error:", err)
	}
}
```

The newline character acts as a message delimiter because TCP itself does not preserve message boundaries.

## TCP Client

```go
package main

import (
	"bufio"
	"fmt"
	"net"
)

func main() {
	conn, err := net.Dial("tcp", "localhost:8080")
	if err != nil {
		panic(err)
	}
	defer conn.Close()

	_, err = conn.Write([]byte("Hello TCP server\n"))
	if err != nil {
		panic(err)
	}

	response, err := bufio.NewReader(conn).ReadString('\n')
	if err != nil {
		panic(err)
	}

	fmt.Print("Server:", response)
}
```

---

## UDP Server

```go
package main

import (
	"fmt"
	"net"
)

func main() {
	address, err := net.ResolveUDPAddr("udp", ":8080")
	if err != nil {
		panic(err)
	}

	conn, err := net.ListenUDP("udp", address)
	if err != nil {
		panic(err)
	}
	defer conn.Close()

	fmt.Println("UDP server listening on :8080")

	buffer := make([]byte, 1024)

	for {
		n, clientAddress, err := conn.ReadFromUDP(buffer)
		if err != nil {
			fmt.Println("read error:", err)
			continue
		}

		fmt.Printf(
			"Received from %s: %s\n",
			clientAddress,
			string(buffer[:n]),
		)

		_, err = conn.WriteToUDP(
			[]byte("Datagram received"),
			clientAddress,
		)
		if err != nil {
			fmt.Println("write error:", err)
		}
	}
}
```

## UDP Client

```go
package main

import (
	"fmt"
	"net"
	"time"
)

func main() {
	serverAddress, err := net.ResolveUDPAddr(
		"udp",
		"localhost:8080",
	)
	if err != nil {
		panic(err)
	}

	conn, err := net.DialUDP("udp", nil, serverAddress)
	if err != nil {
		panic(err)
	}
	defer conn.Close()

	_, err = conn.Write([]byte("Hello UDP server"))
	if err != nil {
		panic(err)
	}

	err = conn.SetReadDeadline(time.Now().Add(2 * time.Second))
	if err != nil {
		panic(err)
	}

	buffer := make([]byte, 1024)

	n, _, err := conn.ReadFromUDP(buffer)
	if err != nil {
		fmt.Println("No response:", err)
		return
	}

	fmt.Println("Server:", string(buffer[:n]))
}
```

The UDP client uses a timeout because UDP does not guarantee that the server will respond.

---

# 20. How to Choose Between TCP and UDP

Ask the following questions.

## Must every byte arrive?

If yes, TCP is usually the better choice.

Examples:

- File transfers
- Database queries
- Payment requests
- Backend APIs

If occasional loss is acceptable, UDP may be considered.

---

## Must the data arrive in order?

If yes, use:

- TCP
- Or a reliable protocol built on UDP, such as QUIC

If order does not matter, UDP may be suitable.

---

## Is late data useless?

If late data is useless, UDP may be appropriate.

Examples:

- Voice calls
- Video calls
- Multiplayer-game position updates
- Live telemetry

---

## Do you need message boundaries?

TCP does not preserve message boundaries.

The application must implement framing.

UDP preserves datagram boundaries.

---

## Do you need multicast or broadcast?

UDP supports multicast and broadcast where the network permits them.

TCP is designed for one-to-one connections.

---

## Are you building ordinary APIs or backend services?

For normal backend services, start with TCP-based protocols such as:

- HTTP/1.1
- HTTP/2
- gRPC
- Database protocols

Use UDP or QUIC only when there is a clear requirement.

---

# 21. Common Misconceptions

## “TCP sends messages”

TCP sends an ordered stream of bytes.

The application is responsible for defining message boundaries.

---

## “UDP packets always arrive”

UDP datagrams may be:

- Lost
- Delayed
- Duplicated
- Reordered

---

## “UDP is always faster”

UDP has lower transport overhead, but the application may need to rebuild many features that TCP already provides.

---

## “TCP guarantees that the application processed the data”

A TCP acknowledgement normally means that the receiving TCP stack received the bytes.

It does not necessarily mean that the application:

- Read the data
- Processed the request
- Updated the database
- Completed the operation successfully

For important operations, application-level acknowledgement is still required.

For example:

```json
{
  "status": "payment_completed",
  "transaction_id": "txn_123"
}
```

---

## “One TCP write produces one network packet”

The operating system may:

- Combine multiple writes
- Split one write into multiple segments
- Buffer data before sending it

Applications should never depend on one `write()` producing one packet or one `read()`.

---

# 22. TCP and UDP From a System-Design Perspective

For backend and system-design interviews, remember:

## TCP is generally selected when:

- Correctness is essential
- Every byte matters
- Data must arrive in order
- Long-lived connections are useful
- The application should not implement reliability itself

Examples:

```text
REST API
gRPC
Database connection
Message broker connection
SSH
File transfer
```

## UDP is generally selected when:

- Low latency is important
- Some packet loss is acceptable
- Old data becomes useless quickly
- The application needs control over retransmission
- Broadcast or multicast is required

Examples:

```text
Video calls
Voice calls
Online games
DNS
Service discovery
QUIC and HTTP/3
```

---

# 23. Final Mental Model

Remember TCP as:

```text
“Deliver this byte stream reliably and in order,
even if retransmission introduces additional delay.”
```

Remember UDP as:

```text
“Send this independent datagram immediately.
My application will decide how to handle loss,
ordering, retries and timing.”
```

Use **TCP** when:

- Correctness matters
- Every byte must arrive
- Ordering matters
- Reliability is required

Use **UDP** when:

- Low latency matters
- Some loss is acceptable
- Recent data is more valuable than old data
- The application needs control over delivery behavior

The most important difference is not simply:

```text
TCP is slow
UDP is fast
```

The correct understanding is:

```text
TCP provides reliability, ordering, flow control
and congestion control.

UDP provides lightweight datagram delivery and
leaves reliability and ordering decisions to the application.
```