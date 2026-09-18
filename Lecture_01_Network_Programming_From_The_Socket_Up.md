# 📡 Lecture 1 — Network Programming: From the Socket Up

> **Course:** Computer Networks (CN at Scaler)
> **Topic:** Network Programming 101 — Bottom-up from sockets to protocol design
> **Instructor Slides:** `CN_at_Scaler___Lesson_1.pdf`

---

## 📚 Course Roadmap (All 8 Weeks)

| Week | Topic |
|------|-------|
| 1 | **Network Programming 101** ← *We are here* |
| 2 | Computer Networking Eagle Eye View |
| 3 | Server Side Considerations for Scale |
| 4 | nginx Deep Dive |
| 5 | Evolution of HTTP |
| 6 | (Ab)using CDNs |
| 7 | Economics of Cloud Tech |
| 8 | Building for Failures |

---

## 🗺️ Lecture Plan (What We Cover Today)

This lecture is built **bottom-up** — starting from the smallest program that can accept a TCP connection, ending with designing your own binary protocol.

1. **A server** — and everything that breaks around it
2. **A client** — plus OSI, DNS, and byte order
3. **curl, many clients at once**, packets on the wire, SSL
4. **Designing a protocol** of your own

---

## 01 · The Server

### 🔑 The Fundamental Insight

> **A TCP server is just seven system calls.**
>
> Everything you have used — Express (Node.js), Flask (Python), `net/http` (Go) — is *a wrapper around these seven calls.*

```
socket → bind → listen → accept → read → write → close
```

| # | Call | Who uses it |
|---|------|-------------|
| 1 | `socket()` | Client + Server |
| 2 | `bind()` | **Server only** |
| 3 | `listen()` | **Server only** |
| 4 | `accept()` | **Server only** |
| 5 | `read()` | Client + Server |
| 6 | `write()` | Client + Server |
| 7 | `close()` | Client + Server |

> 💡 `bind`, `listen`, and `accept` are the calls that **only a server makes**. A client only needs `socket`, `connect`, `read`, `write`, `close`.

---

### 🔨 Echo Server in C (`echo_server.c`)

```c
#include <string.h>
#include <unistd.h>
#include <arpa/inet.h>

int main() {
    int server_fd = socket(AF_INET, SOCK_STREAM, 0);  // Step 1: create socket

    struct sockaddr_in addr = {0};
    addr.sin_family = AF_INET;
    addr.sin_addr.s_addr = INADDR_ANY;  // listen on ALL interfaces
    addr.sin_port = htons(2026);        // our port, in network byte order

    bind(server_fd, (struct sockaddr*)&addr, sizeof(addr));  // Step 2: bind to port
    listen(server_fd, 1);                                    // Step 3: start listening

    while (1) {
        int client_fd = accept(server_fd, NULL, NULL);  // Step 4: accept connection
        char buf[4096];
        int n = read(client_fd, buf, sizeof(buf));      // Step 5: read data
        write(client_fd, buf, n);                       // Step 6: write it back
        close(client_fd);                               // Step 7: close connection
    }
}
```

**Compile and run:**
```bash
gcc -o echo_server echo_server.c
nc localhost 2026        # connect with netcat and type anything
```

> ⚠️ **Limitation:** This version reads **once** and closes. It is not a real conversation. See the persistent version below.

---

### 📖 What Each System Call Does (Deep Dive)

#### `socket(AF_INET, SOCK_STREAM, 0)`
- Creates a **file descriptor** for a TCP socket.
- `AF_INET` → IPv4 address family.
- `SOCK_STREAM` → TCP (reliable, ordered byte stream). Use `SOCK_DGRAM` for UDP.
- Returns an integer (the fd). Like `open()` for files, but for network connections.

#### `bind(server_fd, &addr, sizeof(addr))`
- Attaches the socket to a **specific IP address and port** on your machine.
- `INADDR_ANY` means "listen on every network interface" (all IPs your machine has).
- In production: bind to a **specific** IP when you want deliberate control.

#### `listen(server_fd, backlog)`
- Tells the kernel: *"Start accepting connections on this socket."*
- The second argument (`backlog`) is the **accept queue size** — how many pending connections the kernel holds before refusing new ones.
- The kernel **completes the TCP handshake for you** before `accept()` is called.

#### `accept(server_fd, NULL, NULL)`
- **Blocks** until a client connects, then returns a **new fd** for that client.
- The original `server_fd` stays open to accept more connections.
- Optionally fills in the client's IP/port into the 2nd and 3rd args if you provide them.

#### `read(client_fd, buf, sizeof(buf))`
- Reads up to `sizeof(buf)` bytes from the connection.
- Returns number of bytes read. Returns `0` when the client closed the connection.
- ⚠️ TCP gives you a **byte stream**, not messages — `read()` may return less than a full "message".

#### `write(client_fd, buf, n)`
- Sends bytes to the connected client.
- ⚠️ Writing to a socket whose peer has closed it sends `SIGPIPE` — which **kills your process by default**.

#### `close(client_fd)`
- Sends a **FIN** to the peer (graceful close) and frees the file descriptor.
- `SO_LINGER {1, 0}` sends a **RST** (abortive close) instead.

---

### 🚪 Ports — Why 1024 is the Line in the Sand

```
Port < 1024  → Root (privileged) only
Port ≥ 1024  → Any user can bind
```

**Well-known ports (< 1024):**

| Port | Protocol |
|------|----------|
| 80   | HTTP     |
| 443  | HTTPS    |
| 25   | SMTP     |
| 21   | FTP      |
| 110  | POP3     |

**Common high ports (≥ 1024):**

| Port | Used by |
|------|---------|
| 3000 | Node.js (dev) |
| 8080 | Alternate HTTP |
| 6379 | Redis |
| 2026 | Our examples |

> 🔐 **Security implication:** If you bind port 80, you must run as root. A bug in your request parser then becomes a **root-level compromise**. This is why production servers:
> - Run your app on port **8080** as an unprivileged user
> - Put **nginx** (does almost nothing) on port 80/443 to proxy

---

### 🧠 Thought Experiment — What Breaks When You Skip a Call?

| Skipped Call | Result |
|---|---|
| **No `bind()`** | `Connection refused` — kernel answers client's SYN with a RST immediately. Fast, loud, easy to debug. |
| **`bind()` but no `accept()`** | Connection **hangs silently** — the kernel completes the handshake and puts the connection in the queue. The client waits until TCP times out. Looks exactly like an overloaded server. |

---

### 🔄 Persistent Echo Server (`echo_server_persistent.c`)

The simple version only reads **once** then closes. A real server keeps reading in a loop:

```c
while (1) {
    int client_fd = accept(server_fd, NULL, NULL);
    char buf[4096];
    int n;
    while ((n = read(client_fd, buf, sizeof(buf))) > 0) {  // keep reading
        write(client_fd, buf, n);
    }
    close(client_fd);  // close only when client disconnects (read returns 0)
}
```

---

### 📬 The Accept Queue (`listen(server_fd, backlog)`)

```c
listen(server_fd, 1);  // backlog = 1
```

1. The kernel **finishes the 3-way TCP handshake** before `accept()` returns.
2. Connections pile up in a queue. Queue size = backlog argument.
3. **Full queue behavior is not portable** — some OSes send RST, others let the client hang indefinitely.
4. We passed `1` in the example. **Real servers pass hundreds**.
5. Linux caps the actual value at `net.core.somaxconn` — your code's number may not be what's in effect.

---

### ⚡ Signals — The One That Will Kill You

| Signal # | Name | What It Does |
|---|---|---|
| 2 | `SIGINT` | Ctrl+C — stop process |
| 9 | `SIGKILL` | Cannot be caught or ignored — hard kill |
| 13 | **`SIGPIPE`** | **Write to a closed peer → process dies** |
| 15 | `SIGTERM` | Polite kill — can be caught |

Others: `SIGHUP` on SSH disconnect, `SIGSTOP`/`SIGTSTP` on Ctrl+Z.

> ⚠️ **`SIGPIPE` is the dangerous one.** If the client closes the connection and you try to `write()` again, `SIGPIPE` kills your process silently — no exception, no error log.

**Fix:** Ignore SIGPIPE globally with `signal(SIGPIPE, SIG_IGN)`, then check the return value of `write()` for `-1`.

#### Demo: SIGPIPE in Action

**Server (`sigpipe_server.c`):**
```c
int client_fd = accept(server_fd, NULL, NULL);
char buf[4096];
int n = read(client_fd, buf, sizeof(buf));

sleep(1); // Give client time to force-close

write(client_fd, buf, n); // SIGPIPE fires here → process dies
write(client_fd, buf, n); // Never reached
```

**Client (`sigpipe_client.c`)** — uses RST to abruptly close:
```c
write(fd, "hello", 5);

struct linger l = {1, 0}; // abortive close: RST instead of FIN
setsockopt(fd, SOL_SOCKET, SO_LINGER, &l, sizeof(l));
close(fd);
```

> `SO_LINGER {1, 0}` forces the OS to send a **RST** (reset) instead of **FIN** (graceful close). This simulates a crash or abrupt disconnect.

---

## 02 · You Already Have a Client

> **You do not need to write code to test a server.** Your terminal already has everything you need.

### `telnet` — Type the Protocol Yourself

If a protocol is text-based (HTTP, SMTP, Redis RESP), you can **type it manually** using `telnet`:

```bash
telnet google.com 80       # HTTP
telnet localhost 2026      # our echo server
telnet mail.example.com 25 # SMTP
```

**Demo — talking to Google with 4 lines of typing:**
```
$ telnet google.com 80
Trying 142.251.222.142...
Connected to google.com.
GET / HTTP/1.1
Host: google.com
                            ← blank line (mandatory!)
HTTP/1.1 301 Moved Permanently
Location: http://www.google.com/
Content-Length: 219
...
```

> You are literally speaking HTTP to Google with your keyboard. This shows HTTP is just text over TCP.

---

### Going Beyond Port 80

Most services do not speak HTTP. And once TLS is in the way, `telnet` is useless — it cannot do the TLS handshake. Use:

```bash
# For HTTPS and any TLS-encrypted service:
openssl s_client -connect www.hotstar.com:443

# For raw TCP — connect, listen, or pipe data:
nc localhost 2026     # connect to our echo server
nc -l 9000            # start listening on port 9000
```

| Tool | Use Case |
|------|----------|
| `telnet` | Text protocols, quick testing |
| `openssl s_client` | TLS-encrypted connections |
| `nc` (netcat) | Everything else — raw TCP/UDP |

---

## 03 · Writing the Client

### Client in C (`client.c`)

```c
#include <string.h>
#include <unistd.h>
#include <arpa/inet.h>
#include <netdb.h>

int main() {
    struct hostent *he = gethostbyname("localhost");  // DNS lookup

    int fd = socket(AF_INET, SOCK_STREAM, 0);         // create socket

    struct sockaddr_in addr = {0};
    addr.sin_family = AF_INET;
    memcpy(&addr.sin_addr, he->h_addr_list[0], he->h_length);
    addr.sin_port = htons(8080);

    connect(fd, (struct sockaddr*)&addr, sizeof(addr)); // connect (no bind/listen)

    write(fd, "hello", 5);

    struct linger l = {1, 0};
    setsockopt(fd, SOL_SOCKET, SO_LINGER, &l, sizeof(l));
    close(fd);
    return 0;
}
```

> **Notice:** The client has no `bind()`, `listen()`, or `accept()`. Those are server-only calls.

---

### 🔌 Ephemeral Ports — How Return Traffic Finds the Client

> **Clients never bind. The kernel picks a port for them.**

| Side | Port Behavior |
|------|--------------|
| **Server** | `bind()` to a known port (e.g., 8080) — clients need a fixed address to connect to |
| **Client** | Gets a **random high port** assigned by the kernel at `connect()` time |

- Ephemeral port range on Linux: **32768–60999**
- These ports are **finite**!
- After a connection closes, the port stays in **TIME_WAIT** for ~60 seconds before it can be reused.
- **Busy clients can run out of ephemeral ports** — a real production concern.

---

### 🔄 Byte Order — `htons()` and Friends

> **Your CPU and the network disagree on byte order.**

**Example: Port 2026 = 0x07EA**

| Byte Order | Memory Layout | Used By |
|---|---|---|
| Little-endian | `EA 07` | x86/ARM laptops (your machine) |
| Big-endian (Network order) | `07 EA` | The wire (TCP/IP standard) |

**The conversion functions:**

| Function | Full Name | Use it for |
|---|---|---|
| `htons()` | host to network, short (2 bytes) | Port numbers before sending |
| `htonl()` | host to network, long (4 bytes) | IP addresses before sending |
| `ntohs()` | network to host, short | Port numbers after receiving |
| `ntohl()` | network to host, long | IP addresses after receiving |

> 💡 Always use these when dealing with port numbers and IP addresses. Without them, port `2026` on your x86 machine would go out as `EA 07` instead of `07 EA` — and the receiver would see port 64007!

---

### 🌐 DNS — The Secret Inside `gethostbyname()`

```c
struct hostent *he = gethostbyname("localhost");
```

> **One function call is a whole protocol.** This is DNS hidden inside a synchronous blocking call.

**What happens when you call `gethostbyname("google.com")`:**

```
Step 1: Check /etc/hosts  ← no network, instant
        If found → return IP immediately

Step 2: Send UDP query to DNS resolver (/etc/resolv.conf)
        Resolver recurses: Root → TLD (.com) → Authoritative (google.com)
        Returns IP address

Step 3: Cache result for the TTL duration
        Fast when cached, can take seconds when cold — with no timeout you control
```

**Why this matters:**
- `/etc/hosts` **overrides DNS** — editing it overrides everything (useful debugging trick)
- It **blocks** your thread with no timeout you can control
- ⚠️ `gethostbyname()` is **IPv4-only** and **not thread-safe** → use `getaddrinfo()` in real code

---

### 🧱 OSI Model — Where Our Code Lives

> We have been writing layer 7 code this whole time, touching layers 3 and 4 without naming them.

```
Layer 7  Application   ← Our echo protocol, HTTP
Layer 6  Presentation
Layer 5  Session
Layer 4  Transport     ← TCP, ports, htons(2026)
Layer 3  Network       ← IP, INADDR_ANY, 127.0.0.1
Layer 2  Data Link     ← Ethernet, Wi-Fi, MAC addresses
Layer 1  Physical      ← Copper, fibre, radio waves
```

**In our 20-line C programs:**
- We used **IP addresses** (Layer 3)
- We used **ports** (Layer 4)
- We sent **data** (Layer 7 — our "protocol" echoes bytes)

The layers below (MAC, Ethernet, physical wire) are handled entirely by the OS and hardware. We never see them.

---

## 04 · HTTP & curl

### HTTP is Just `client.c` with a String in It

HTTP/1.1 is a **text protocol over TCP port 80**. An HTTP request looks like:

```
GET / HTTP/1.1\r\n
Host: google.com\r\n
\r\n
```

Two important rules:
1. **CRLF line endings** (`\r\n`), not just `\n`
2. A **mandatory blank line** after headers — signals end of headers to the server

Point your C client at port 80, write this string, read until close. That is the whole protocol.

### `curl` — The Best HTTP Teaching Tool

```bash
curl google.com                          # basic request
curl -i google.com                       # show response headers + body
curl -vv https://google.com 2>&1 | less  # verbose: see TLS handshake + headers + body
```

| Flag | What it shows |
|------|--------------|
| (none) | Just the response body |
| `-i` | Response headers + body |
| `-v` / `-vv` | Everything: connection, TLS handshake, request headers, response |

> `curl -vv` is your best friend for debugging HTTP. Every abstraction you use is doing exactly what `-vv` shows you.

---

## 05 · Many Clients at Once

### The Problem

Our echo server serves **exactly one client at a time**:
```c
while (1) {
    int client_fd = accept(...);  // blocks until one client connects
    // handle that single client
    close(client_fd);
    // only now can we accept the NEXT client
}
```

### Two Solutions (Both Used in Production Today)

#### Solution 1: One Process Per Connection — `fork()`

```c
while (1) {
    int client_fd = accept(server_fd, NULL, NULL);

    if (fork() == 0) {           // child process: fork() returns 0 in child
        char buf[4096];
        int n;
        while ((n = read(client_fd, buf, sizeof(buf))) > 0) {
            write(client_fd, buf, n);
        }
        close(client_fd);
        return 0;                // child exits when done
    }

    close(client_fd);            // parent closes its copy and loops back to accept()
}
```

**`fork()` return values:**

| Return value | You are... | What to do |
|---|---|---|
| `0` | The **child** | Handle this client, then exit |
| `> 0` | The **parent** | That number is child's PID. Close your fd copy, go back to `accept()` |
| `-1` | fork failed | Out of memory or processes. Almost nobody checks this. |

**Pros:** Simple, crash-isolated (one child crash does not affect others), OS handles scheduling.

**Cons:** One **process** per client. Fine for hundreds of clients, completely hopeless at ten thousand (C10K problem).

---

#### Solution 2: One Process, Many Sockets — `select()` / `epoll`

> Ask the kernel which sockets are ready, then handle only those.

Instead of blocking on one socket, you hand the kernel a *set* of fds and it tells you which ones have data.

| Mechanism | Portability | Time Complexity | How it works |
|---|---|---|---|
| `select()` | Works everywhere | O(n) — scans all fds | Rebuild the fd set on every call; kernel walks all descriptors |
| `epoll` | Linux only | O(ready) — only active | Register once; kernel returns only changed descriptors |
| `kqueue` | BSD + macOS | O(ready) | macOS equivalent of epoll |
| `IOCP` | Windows only | O(ready) | Windows equivalent |

> **`epoll` is what nginx and Node.js are built on.** It is why they can handle millions of concurrent connections efficiently.

**The trade-off of epoll:** One slow handler blocks everything in the same process. Unlike `fork()`, there is no crash isolation.

---

### Client-Side Limits

The problem is not just server-side. The client has limits too:

| Limit | What it means |
|---|---|
| **Chrome per-host connection cap (historically 6)** | Requests 7–10 simply queue up. This is why HTTP/2 multiplexing exists. |
| **`ulimit -n`** | Hard ceiling on open file descriptors per process. Sockets count toward this limit. Often just 1024. |
| **Connection pools** | Open once, reuse many times. Skips the TCP and TLS handshake on every reuse after the first. |

---

### Socket Options

```c
setsockopt(server_fd, SOL_SOCKET, SO_REUSEADDR, &opt, sizeof(opt));
```

| Option | What it does |
|---|---|
| `INADDR_ANY` | Listen on every network interface. Convenient in dev, a deliberate choice in prod. |
| `SO_REUSEADDR` | Allow restarting your server without "address already in use" error. Without it, old socket in TIME_WAIT blocks the port. |
| Promiscuous mode | See traffic not addressed to you. How Wireshark and packet sniffers work. Why plaintext protocols died. |

> `SO_REUSEADDR` is the one you will actually use in every real server.

---

### 🔍 Seeing the Actual Bytes — `tcpdump` and Wireshark

> **Stop guessing. Look at the bytes.**

```bash
# Capture traffic on port 2026, save to file
sudo tcpdump -ni any port 2026 -w 1.pcap

# Read captured traffic as ASCII
tcpdump -r local_capture.pcap -A

# Read as hex + ASCII side by side
tcpdump -r local_capture.pcap -X
```

Capture it once, then read it as ASCII and as hex. Everything we have talked about — headers, length prefixes, connection states — is visible in there.

`Wireshark` is the GUI version with better filtering and protocol dissectors.

---

### 🔐 SSL / TLS

> **SSL secures TCP, not HTTP.** It sits *under* the application protocol.

The same TLS machinery protects HTTP (HTTPS), SSH, SMTP over TLS, database connections, and anything else you put on a socket.

**How TLS works:**

```
Step 1: Key Exchange (Asymmetric crypto)
        Client and server agree on a shared secret
        Uses RSA or elliptic curve — slow but necessary

Step 2: Symmetric Encryption for Data
        Use the shared secret with AES or ChaCha20
        Symmetric is dramatically faster than asymmetric

Step 3: Certificate Verification
        Server sends certificate (public key + identity)
        Certificate is signed by a Certificate Authority (CA)
        CA chain is why you believe the server is who it claims to be
        This is the certificate that expires at 3am and takes your site down
```

**Certificate Pinning:**
- Removes the CA from the trust equation
- "Trust **this exact certificate**, not whatever any CA in the world is willing to sign"
- Common in mobile apps for extra security against MITM attacks

---

## 06 · Protocol Design

### Text vs Binary — The Trade-off

> HTTP proves how far human-readable can take you. HTTP/2 is binary.

| | TEXT | BINARY |
|---|---|---|
| **Readability** | ✅ You can read it — telnet in, tcpdump it, eyeball logs | ❌ Need tooling to see anything |
| **Ambiguity** | ❌ Verbose, edge cases, case sensitivity, whitespace rules | ✅ Compact, unambiguous |
| **Performance** | ❌ Slower to parse | ✅ Cheap to parse |
| **Debugging** | ✅ Easy | ❌ Requires hex/base64 fluency |
| **Examples** | HTTP/1.1, Redis RESP, SMTP | HTTP/2, gRPC/protobuf, TLS |

> HTTP won *because* it was human-readable. HTTP/2 went binary because the text overhead was a real cost at scale.

---

### 📦 Message Framing — The Most Important Decision

> **TCP gives you a byte stream, not messages.**
>
> If you send two messages back-to-back, `read()` on the other side may return them merged, split, or in any combination. You must decide: how does the receiver know a message ended?

**There are only 3 answers:**

#### 1. Fixed Length — Every message is exactly N bytes

```
[NNNNNNNNNNNNNNNN]  ← always exactly N bytes
```
- ✅ Trivial to parse, zero ambiguity
- ❌ Padding waste for shorter messages
- ❌ Cannot change N after deployment

#### 2. Delimited — Read until a sentinel character

```
hello\n
world\n
```
- ✅ Simple — HTTP uses a blank line `\r\n\r\n` between headers and body
- ❌ Problem: what if the delimiter appears *inside* your data? You need escaping.
- ❌ Escaping logic is complex and error-prone

#### 3. Length-Prefixed — Send the size, then the bytes

```
[4-byte length][... body bytes ...]
```
- ✅ Robust — read exactly `length` bytes and stop
- ✅ Binary-safe (no sentinel conflicts)
- ✅ What most modern protocols use
- HTTP uses this too: `Content-Length: 219`

> **When in doubt, use length-prefixing.** It is robust, binary-safe, and what most modern protocols do.

---

### 🔢 BCD — Binary Coded Decimal

> Four bits (one nibble) per decimal digit.

```
Decimal:  9    8    7    6    5
Nibbles: 1001 1000 0111 0110 0101
Hex:      98   76   5F
```

**Properties:**
- Half the size of ASCII for numeric data
- No conversion step needed
- Exact decimal arithmetic (no floating-point errors)
- Still used in SIM cards, SMS, and credit card transactions today

---

### 📝 ASN.1 — Write the Grammar, Generate the Codec

> The most powerful serialization system — describe the shape of your data once, and encoder/decoder are generated automatically.

```asn1
MyModule DEFINITIONS ::= BEGIN
    Item ::= SEQUENCE {
        itemCode    INTEGER (1..99999),
        color       VisibleString ("Black" | "Blue" | "Brown"),
        isTaxable   BOOLEAN
    }
END
```

- You write the **schema** once in a language-neutral grammar
- Tools **generate** encoder and decoder — you never hand-write parsing
- Created in **1984** and still powering X.509 certificates, LDAP, SNMP, and the entire telecom stack today

---

### 🏷️ TLV — Tag-Length-Value

> ASN.1 you can write on a napkin.

```
[Type][Length][Value][Type][Length][Value]...
```

Every field is:
1. **Type** — what kind of data is this?
2. **Length** — how many bytes follow?
3. **Value** — the actual data bytes

**The killer property:** A parser that has **never seen this field type** can still **skip it** — just jump forward by `Length` bytes. This makes TLV inherently **forward-compatible**. Old parsers gracefully handle new fields.

**Used in:** TLS extensions, ISO 8583 (payment messages), Protocol Buffers (protobuf), EMV chip cards, TLS record layer.

---

### 🔑 base64 and hex — How You Debug Binary

#### HEX — For Reading

- **2 characters per byte**
- `tcpdump -X` shows hex alongside ASCII
- Fluency here means spotting a length prefix or a repeated tag by eye

```
48 65 6C 6C 6F  →  Hello
```

#### BASE64 — For Transport

- Survives text-only channels: HTTP Basic Auth, JWTs, email attachments, `data:` URIs
- **33% overhead** — 3 bytes become 4 characters
- A transport wrapper, not a serialization format

```
Hello  →  SGVsbG8=  (base64)
```

---

### 🔧 RPC — Remote Procedure Call

> **RPC is framing plus a function name.**

Send the function name + arguments, get a response back.

The terms:
- **Marshalling** — objects → bytes (encoding the call to send)
- **Demarshalling** — bytes → objects (decoding the response received)

**How it works:**
1. **Marshal** — function name + arguments → bytes (using your encoding choice)
2. **Frame and send** — length-prefix it, send it over the socket
3. **Receive and unmarshal** — read the response, decode it

> ⚠️ **The network is not a function call.** Calls can fail halfway, time out with the work possibly already done, and get retried causing duplicates. Every RPC framework rediscovers this the hard way.

---

### ⚡ gRPC — RPC with protobuf

> Much faster to encode/decode than JSON, much easier to understand than ASN.1. That combination is why it won.

| Component | What it does |
|---|---|
| `.proto` file | Schema-first. Define types once, generate client + server code in any language |
| Wire format | **TLV** (Tag-Length-Value) — new fields do not break old clients |
| Transport | **HTTP/2** — binary and multiplexed (many streams over one TCP connection) |
| Trade-off | No `curl` — you cannot read it without tooling. Speed bought with debuggability. |

**Flow:**
```
.proto schema
    ↓ protoc codegen
Client stub ←→ gRPC server
    ↓ wire
TLV bytes over HTTP/2 over TLS over TCP
```

---

## 📝 Homework — Things to Try

1. **Wireshark on your laptop** — If you use Redis, look for its traffic. RESP is text and length-prefixed — you will recognize it immediately.

2. **Write a server and client** in your language — the system calls have the same names across languages. That is the point of starting in C.

3. **Make them do more than echo** — The moment there is structure, you must pick a framing strategy. Notice which one you reach for naturally.

4. **Write base64 encoding from scratch** — Forces you to think in bit groups, not just bytes. Exactly the mental model binary protocol work needs.

5. **Pass complicated messages with TLV** — Both directions, nested structures, optional fields. Add a field and check your old parser still works (forward compatibility test).

6. **Write a client that imitates `curl`** — Follow redirects, print headers, handle chunked encoding. You will end up actually reading the RFC.

---

## 🔑 Key Takeaways

```
socket · bind · listen · accept · read · write · close
```

> Every abstraction you use sits on these seven calls. Go find the `accept()` inside your framework.

| Concept | What to Remember |
|---|---|
| TCP server = 7 syscalls | Everything else is a wrapper |
| Ports < 1024 | Root only — binding port 80 means running as root |
| `listen(fd, backlog)` | Kernel queues and completes handshakes before `accept()` |
| `SIGPIPE` | Write to closed peer = silent process death — ignore it or handle it |
| Clients do not bind | Kernel assigns ephemeral port from range 32768–60999 |
| `htons()` / `htonl()` | Convert between host and network byte order |
| `gethostbyname()` | DNS hidden in a blocking call — use `getaddrinfo()` in real code |
| OSI Layers 3/4/7 | IP / TCP+ports / application — all touched in 20 lines of C |
| HTTP = text over TCP | `curl -vv` shows you everything |
| `fork()` | Simple concurrency — one process per client |
| `epoll` | Efficient concurrency — O(ready), what nginx and Node.js use |
| `SO_REUSEADDR` | Avoid "address already in use" on server restart |
| Message framing | Fixed / Delimited / Length-prefixed — pick one, be deliberate |
| TLV | Forward-compatible binary framing used by TLS, protobuf, ISO 8583 |
| gRPC | protobuf schema + TLV wire + HTTP/2 transport |

---

*Notes generated from: `CN_at_Scaler___Lesson_1.pdf` · Lecture 1 of 8*
