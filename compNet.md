# Computer Networks – 2-Hour Tutor Session Script

*(“What actually happens when you open a website?”)*

> This script is written so you can literally read it out loud.
> Things in **bold** are lines you can say.
> Things in *italics or [brackets]* are notes to yourself.

---

## 0. Before You Start (for you, not to read)

* Draw a simple diagram on the board:

  * A **laptop** → **WiFi/router box** → **cloud (Internet)** → **server**.
* Keep colored pens if possible:

  * 1 color for **hosts/devices**,
  * 1 for **addresses**,
  * 1 for **protocols** or **layers**.
* Your mental mission:

  > “Tell one clean story of a packet’s life, and name all the concepts the exam cares about along the way.”

---

## 1. Hook & Warm-Up (5–10 minutes)

**You:**
“Alright, let’s start with something super simple:

When you open your browser, type `google.com`, and hit Enter…
What do you think actually happens behind the scenes?”

*Pause. Let them answer. Accept anything – “server”, “request”, “DNS”, even “magic.”*

**You:**
“There’s no grading on answers here. Just throw out ideas:

* Does my laptop talk directly to Google?
* Does it go to my router first?
* Does it talk to some mystery thing in my ISP?
* What do you think?”

*Take 3–5 answers. Smile, nod, don’t correct yet.*

**You:**
“Cool. So today, I want you to walk out of here able to tell a very specific story:

> ‘I type a URL, my laptop does A, then B, then C, it talks to my router like this, switches do that, routers do this, DNS answers this question, and that’s how the page shows up.’

We’re going to follow **one packet** on its journey across the network and back, and on the way we’ll touch:

* Hosts, MAC addresses, IP addresses
* Networks and default gateway
* Hub, switch, and router
* The OSI model (especially Layers 1–4)
* Protocols like ARP, DNS, DHCP, HTTP, TCP, UDP
* And a bit of router hierarchy and summarization
* Plus the top layers: application-level protocols like HTTP, DNS, etc.

Most final-exam questions about networking are really just different angles on that one story.”

---

## 2. Hosts, MAC, IP, Network, Default Gateway (15–20 minutes)

[**Draw**: laptop, phone, maybe a printer, all connected to a WiFi/router box. Label the box “WiFi Router”.]

**You:**
“Let’s start with the basics: **who** is talking on the network, and **how** do we name them?

In networking we call any device that can send and receive data a **host**:

* Your laptop is a host.
* Your phone is a host.
* A server in some data center is also a host.

Every host has at least two important ‘identities’ in networking:

1. A **MAC address**
2. An **IP address**”

### 2.1 MAC Address – Local Identity

[Write: `MAC address = physical identifier (Layer 2)`]

**You:**
“A **MAC address** is like a physical ID that’s burned into the network card at the factory.

* It looks like `AA:BB:CC:DD:EE:FF`.
* It’s usually **unique** per network interface.
* It’s used **on the local network**, especially by **switches**.

Analogy time:

> ‘MAC address is like your **face** – people right around you use it to recognize you.’”

Ask them:

**You:**
“Has anyone here ever run `ipconfig /all` on Windows or `ifconfig` / `ip a` on Linux and seen those weird hex numbers? That’s usually your MAC address.”

### 2.2 IP Address – Global (Logical) Identity

[Write: `IP address = logical address (Layer 3)`]

**You:**
“An **IP address** is a **logical** address.

* IPv4 looks like `192.168.1.10` or `8.8.8.8`.
* IPv6 is the longer one, like `2001:db8::1`.
* IP addresses can change: you can move networks and get a new IP.

Analogy:

> ‘IP address is like your **street address** – it tells the world where to send stuff.’”

**You:**
“MAC exists down close to the wire, IP exists when you’re thinking about different networks talking to each other.”

### 2.3 What Is a “Network”?

[Write: `Network = group of IPs that can talk directly (no router needed)`]

**You:**
“When we say ‘a network’ in IP terms, we usually mean:

> ‘A group of IP addresses that can reach each other directly, without needing a router.’

Example:

* Let’s say your devices at home are:

  * Laptop: `192.168.1.10`
  * Phone: `192.168.1.11`
  * Printer: `192.168.1.50`
* They all share:

  * IPs that start with `192.168.1.*`
  * And a subnet mask like `255.255.255.0`

We say they are on the **same network**, `192.168.1.0/24`.

Inside that network, they can send data to each other directly. No router needed.”

### 2.4 Default Gateway – The Exit Door

[On the router box, write: `192.168.1.1 (default gateway)`]

**You:**
“Now, what if your laptop wants to reach a server somewhere on the internet, like `142.250.191.46` (one of Google’s real IPs)?

That IP is not in your `192.168.1.*` network.

So your laptop thinks:

> ‘This destination is out of my local neighborhood. I need to go to the **exit door**.’

That exit door is the **default gateway**.

* The default gateway is usually your home **router**.
* Its IP inside your network is something like `192.168.1.1`.
* Your laptop is configured with:

  * Its own IP
  * Subnet mask
  * Default gateway
  * DNS server

So rule of thumb:

> ‘If destination IP is in **my** network → talk directly.
> If not → send it to **default gateway**.’”

### 2.5 Quick Check

**You:**
“Let’s check if this is clear. Answer in your own words:

1. What’s the difference between a **MAC address** and an **IP address**?
2. What is a **network** in this context?
3. What is a **default gateway**?”

*Take a few answers. Clarify gently.*

---

## 3. Devices: Hub, Switch, Router (15–20 minutes)

[Draw three boxes: HUB, SWITCH, ROUTER. Connect them crudely with lines.]

**You:**
“Now let’s talk about three classic devices:

* **Hub**
* **Switch**
* **Router**

You will almost certainly see these on exams.”

### 3.1 Hub – The Shouter

**You:**
“A **hub** is basically a multi-port repeater.

* It doesn’t understand addresses.
* Whatever comes in one port gets **blasted out** to all ports.
* Every device plugged into a hub sees all the traffic.

Analogy:

> ‘A hub is like someone shouting in a room. Everyone hears everything, even if it’s not for them.’

This is **inefficient** and **not secure** (everyone sees everyone’s frames), which is why hubs are basically not used anymore in modern networks.”

### 3.2 Switch – The Smart Phonebook

[Under SWITCH, write: `MAC table / CAM table`.]

**You:**
“A **switch** is smarter.

* It lives mostly at **Layer 2** (Data Link).
* It learns a **MAC address table**:

  * ‘MAC X is on port 1’
  * ‘MAC Y is on port 3’

When a frame arrives:

* It looks at the **destination MAC**.
* If it knows which port that MAC is on, it sends the frame **only there**.
* If it doesn’t know, it floods it out all ports (like a hub) – then learns from the reply.

Result:

* Less noise.
* Better performance.
* Only the intended recipient sees most traffic.”

Analogy:

> “Imagine a receptionist who first shouts every name out loud,
> but gradually builds a list: ‘Person A sits at desk 4, Person B at desk 7.’
> Eventually, they stop shouting and direct visitors straight to the right desk.”

### 3.3 Router – The Border Guard / GPS

[Under ROUTER, write: `Layer 3, Routing Table`. Draw two “networks” on each side.]

**You:**
“A **router** connects **different IP networks**.

* It lives mostly at **Layer 3**.
* It doesn’t care about MAC addresses long-term; it cares about **IP networks**.
* It has a **routing table** – a list of:

  * Networks
  * And where to send packets to reach them (next hop, interface).

When a packet comes in, the router:

1. Looks at the destination IP.
2. Finds the **best matching route** in its table.
3. Sends the packet out through the corresponding interface, maybe to another router.

Analogy:

> ‘A router is like a GPS or a border guard. It decides where to send traffic next based on the destination address.’

So summary:

* **Hub**: repeats bits, no brain.
* **Switch**: forwards based on **MAC** inside a LAN.
* **Router**: forwards based on **IP** between networks.”

### 3.4 Quick Check

**You:**
“Try to finish these sentences:

* A **hub** is like…
* A **switch** is like…
* A **router** is like…”

*Take a few analogies, then give your clean version if needed.*

---

## 4. OSI Model – Especially Layers 1–4 (20–25 minutes)

[Draw OSI stack with 7 boxes: 1 at bottom, 7 at top.]

**You:**
“Now let’s talk OSI.

The **OSI model** is just a way to break the job of networking into **7 layers**.
Each layer has a job, and data passes down the stack on the sender and up the stack on the receiver.

I’ll go quickly through all 7 but focus more on **Layers 1–4**.”

### 4.1 Layers Overview

**You:**
“From top (closest to the app) to bottom (closest to the wire):

7. **Application** – what users and apps see (HTTP, DNS, etc.)
8. **Presentation** – data format, encoding, sometimes encryption
9. **Session** – start/maintain/end conversations
10. **Transport** – TCP/UDP, ports, reliability
11. **Network** – IP, routing between networks
12. **Data Link** – frames, MAC, Ethernet, switches
13. **Physical** – bits, signals, cables, radio waves”

*Maybe write a mnemonic if they use one in class (like “All People Seem To Need Data Processing”).*

### 4.2 Layer 1 – Physical

**You:**
“Layer 1 is **Physical**.

* This is the raw medium: electrical signals, light, radio.
* Cables (Ethernet, fiber), Wi-Fi radio waves, connectors, voltages.
* Questions here: is the bit 0 or 1? Is there a signal or not?

Devices here are things like: repeaters, some parts of NIC hardware.”

### 4.3 Layer 2 – Data Link

**You:**
“Layer 2 is **Data Link**.

* Units here are **frames**.
* It deals with **MAC addresses**.
* It handles things like:

  * Framing data
  * Detecting some errors
  * Local delivery inside a single network segment

Ethernet is a classic Layer 2 technology.

Switches are **mostly Layer 2** devices.”

### 4.4 Layer 3 – Network

**You:**
“Layer 3 is **Network**.

* Units here are **packets**.
* It uses **IP addresses**.
* It decides how to get from one network to another (routing).

Routers live here.

Protocols:

* IPv4, IPv6, ICMP, etc.”

### 4.5 Layer 4 – Transport (TCP vs UDP)

**You:**
“Layer 4 is **Transport**.

* It provides **end-to-end communication** between applications on different hosts.
* It uses **ports**, like port 80, 443, 53, etc.
* Two main protocols:

  * **TCP** – reliable, connection-oriented
  * **UDP** – faster, connectionless, no built-in reliability.”

Go a little deeper:

**You:**
“**TCP**:

* Before sending data, it does a **3-way handshake** (SYN, SYN-ACK, ACK).
* It guarantees:

  * Data is delivered
  * In order
  * Without duplicates
* If packets are lost, it retransmits.
* Used for: web browsing (HTTP/HTTPS), email, file transfer, most APIs.

**UDP**:

* No handshake
* No guarantee data will arrive
* No guarantee of order
* Much less overhead
* Used for: DNS queries, streaming, gaming, voice/video calls, some VPNs.

So one classic exam question:

> ‘Why would real-time apps like voice and video often use UDP instead of TCP?’
> Because they prefer **speed and low latency** over 100% reliability.
> Losing a little audio is better than freezing.”

### 4.6 Quick OSI Check

**You:**
“Let’s map devices to OSI layers quickly:

* Switch lives mainly at Layer ___?
* Router lives mainly at Layer ___?
* TCP and UDP live at Layer ___?

Answers:

* Switch → **Layer 2**
* Router → **Layer 3**
* TCP/UDP → **Layer 4**”

---

## 5. The Packet’s Journey: From Browser to Server and Back (25–30 minutes)

Now we glue everything together.

On the board, write a numbered list from 1 to maybe 9 or 10, and fill as you speak.

### 5.1 Step 0: Get an IP (DHCP)

**You:**
“Before your laptop can talk to anybody, it needs a valid **IP configuration**.

This is usually handled by **DHCP** – Dynamic Host Configuration Protocol.

When you connect to Wi-Fi:

1. Your laptop sends a broadcast:

   > ‘Hey, is there a DHCP server? I need an IP address.’
2. The DHCP server (often your router) responds with:

   * Your IP address
   * Subnet mask
   * Default gateway
   * DNS server

That’s how your laptop gets something like:

* IP: `192.168.1.10`
* Mask: `255.255.255.0`
* Gateway: `192.168.1.1`
* DNS: `192.168.1.1` or `8.8.8.8`

You never see it; it just happens.”

### 5.2 Step 1: You Type `google.com`

**You:**
“You open your browser and type `google.com`. You hit Enter.

Your computer needs to turn that **name** into an **IP address**.”

### 5.3 Step 2: DNS – Name to IP

**You:**
“This is where **DNS** comes in – Domain Name System.

It’s basically the **phonebook of the internet**.

* Your device sends a DNS query:

  > ‘What is the IP address of `google.com`?’
* The DNS server responds with the IP, something like `142.250.xxx.xxx`.

Now you know **where** to send packets.”

You can add:

**You:**
“DNS usually uses **UDP on port 53**, by the way – good exam fact.”

### 5.4 Step 3: Check If Destination Is Local or Remote

**You:**
“Now your laptop asks:

> ‘Is this destination IP in my own network?’

It uses its own IP + subnet mask to figure out the **network address**.

* If the destination is in the same network, it sends directly.
* If it’s not, it knows:

  > ‘Send it to my **default gateway**, the router.’”

For a final, you don’t necessarily need to do binary math here unless your course expects it.

### 5.5 Step 4: ARP – Find the Router’s MAC

**You:**
“To send a frame to the router, your laptop needs the router’s **MAC address**.

That’s where **ARP** – Address Resolution Protocol – comes in.

ARP works like this:

1. Your laptop broadcasts:

   > ‘Who has IP `192.168.1.1`? Tell `192.168.1.10`.’
2. The router replies:

   > ‘I do. My MAC is AA:BB:CC:DD:EE:FF.’

Now your laptop can build a **Layer 2 frame**:

* Destination MAC = router’s MAC
* Source MAC = laptop’s MAC
* Inside it, there’s a **Layer 3 IP packet** going to Google’s IP
* Inside that, there’s **Layer 4 TCP data** going to port 443 (HTTPS).”

You can say:

> “ARP glues **IP addresses** (Layer 3) to **MAC addresses** (Layer 2) on the local network.”

### 5.6 Step 5: Host → Switch

**You:**
“The frame leaves your laptop and travels over **Layer 1** (radio waves if Wi-Fi, cable if Ethernet) to the **switch**.

The switch:

* Looks at the **source MAC** and updates its MAC table:

  > ‘MAC of this laptop is on port 2.’
* Looks at the **destination MAC** (router’s MAC).
* Forwards the frame only out the port where the router is connected.”

So here, **Layer 2** is doing its job.

### 5.7 Step 6: Switch → Router

**You:**
“The router receives the frame.

* It strips off the **Layer 2 (Ethernet) header**.
* It looks at the **Layer 3 IP header**:

  * ‘Destination IP = Google’s IP.’

The router then checks its **routing table**:

* Maybe it has:

  * `192.168.1.0/24 → local LAN`
  * `0.0.0.0/0 → send to my ISP`

The default route (`0.0.0.0/0`) basically means:

> ‘For anything I don’t know specifically, send it to the ISP.’

So the router sends the packet out its **WAN interface** toward the ISP.”

### 5.8 Step 7: Across the Internet – Multiple Routers

[Draw 3–5 routers between Home and Server.]

**You:**
“On the internet, the packet passes through many routers.

Each one does the same basic process:

1. Receive the packet.
2. Look at the destination IP.
3. Check its routing table.
4. Forward it to the next hop.

This might happen 5, 10, 15 times before reaching Google’s network.

If you’ve ever used `traceroute` or `tracert`, you’ve seen those hops:

* Each line is one router along the path.”

### 5.9 Step 8: Destination Network & Server

**You:**
“Finally, the packet reaches the destination network:

* It hits Google’s edge router.
* That router routes it into Google’s internal network.
* Eventually it reaches the **specific server** handling your request.

On the last hop, ARP is used again:

* The last router ARPs for the server’s MAC.
* Then frames are sent on that LAN to the server.”

### 5.10 Step 9: TCP & Application Response

**You:**
“Remember, at Layer 4 this is a **TCP connection**:

* Your computer opened a connection from:

  * Source IP: your IP
  * Source port: some random high port (e.g. 50321)
  * Destination IP: server IP
  * Destination port: 443 (HTTPS)

TCP did a **3-way handshake**:

* SYN → SYN-ACK → ACK

Then HTTP/HTTPS (Layer 7) sends:

* ‘GET / HTTP/1.1’
* Headers, cookies, etc.

The server responds:

* HTTP status code
* HTML, CSS, JS, etc.

That data travels back through the same layers, in reverse, to your browser, which renders the page.”

**You:**
“So, that long story we just told:

* Starts at your app (browser).
* Goes down through Layers 7 → 1 on your side.
* Crosses the network.
* Goes up from 1 → 7 on the server.
* And back.

And along the way, we met: DHCP, DNS, ARP, switches, routers, IP, MAC, TCP, HTTPS, OSI layers…
That’s the core of computer networking.”

---

## 6. Router Hierarchies & Route Summarization (🟡, 10–15 minutes)

[Draw three layers of routers: Access at bottom, Distribution in middle, Core at top.]

**You:**
“Now let’s zoom out and talk about **big networks**, like an ISP or a big company.

You don’t just have one router. You have many routers arranged in a **hierarchy**.

Common design:

* **Access layer**: routers/switches closest to users.
* **Distribution layer**: aggregates multiple access networks.
* **Core layer**: very fast backbone in the center.

If every router had to store routes for every tiny subnet individually, routing tables would be huge and slow. That’s where **route summarization** comes in.”

### 6.1 Summarization Intuition (No Heavy Math)

**You:**
“**Route summarization** (also called **route aggregation**) means:

> ‘Combine many specific routes into one more general route.’

Example conceptually:

* Instead of storing:

  * `192.168.1.0/24`
  * `192.168.2.0/24`
  * `192.168.3.0/24`
* You might summarize them as:

  * `192.168.0.0/22` (one route covering all of them)

Why do this?

* Smaller routing tables → less memory.
* Faster lookups.
* Simpler management.

So a **higher-level router** might just know:

> ‘To reach anything in 192.168.0.0/22, send it this way.’

It doesn’t care about each little subnet; the **lower-level routers** handle the details.”

You can add:

**You:**
“For exams, usually they care that you know **what** summarization is and **why** it’s used, not that you memorize binary prefixes—unless your course specifically drills that.”

---

## 7. OSI Layers 5–7 & Application Protocols (🟡, 10–15 minutes)

Now come back to the OSI model and focus on the top.

[Circle Layers 5, 6, 7 on your earlier stack.]

### 7.1 Layer 5 – Session

**You:**
“**Layer 5 – Session**.

* Manages **sessions** between applications.
* Think of:

  * Logging into a remote server.
  * Maintaining a session for a VoIP call.
  * Managing when a session starts, keeps going, and ends.

In practice, many real-world implementations blur the lines, but conceptually, this is session management.”

### 7.2 Layer 6 – Presentation

**You:**
“**Layer 6 – Presentation**.

* Deals with **formatting**, **encoding**, sometimes **compression** and **encryption**.
* Examples:

  * Converting between character sets.
  * Serializing data (like JSON vs XML).
  * Encryption protocols like **TLS** are conceptually mapped around here (often treated as between Layer 4 and 7 in real stacks).

So when people talk about:

* Converting data to a standard format
* Encrypting before sending

they’re in Presentation-land.”

### 7.3 Layer 7 – Application

**You:**
“**Layer 7 – Application**.

This is where the **protocols that applications use** live – the stuff users and developers actually know by name:

* **HTTP/HTTPS** – web browsing, REST APIs
* **FTP / SFTP** – file transfer
* **SMTP, IMAP, POP3** – email
* **DNS** – domain name lookups
* **DHCP** – handing out IP configs
* **SSH** – secure remote login

These protocols sit on top of TCP or UDP and express the **meaning** of the communication:

* Are we requesting a web page?
* Are we sending an email?
* Are we resolving a domain name?”

### 7.4 Quick Mapping Exercise

**You:**
“Let’s map a few protocols to OSI layers:

* DNS → which layer?

  * Conceptually it’s an **Application layer** protocol, though it uses UDP/TCP at Layer 4.
* HTTP/HTTPS → Application.
* DHCP → Application (using UDP underneath).
* TLS (for HTTPS) → often mapped at Presentation / between Transport and Application.

For exam answers, they usually want:

> DNS, HTTP, FTP, SMTP = **Layer 7** application protocols.”

---

## 8. Final Exam Style Questions & Recap (15–20 minutes)

Now you shift into **exam-prep mode**.

### 8.1 Rapid-Fire Concept Questions

**You:**
“Alright, let’s do some exam-style questions. Answer out loud if you can, or at least answer in your head.”

You can ask:

1. **OSI & Devices**

   * “At which OSI layer does a **switch** operate?”
     → Layer 2.
   * “At which OSI layer does a **router** operate?”
     → Layer 3.
   * “At which OSI layer do **TCP and UDP** operate?”
     → Layer 4.

2. **Addresses**

   * “What is the main difference between a **MAC address** and an **IP address**?”
     → MAC is physical/local, Layer 2; IP is logical/global, Layer 3.
   * “Why do we need both MAC and IP?”
     → MAC for local delivery, IP for inter-network routing.

3. **Protocols**

   * “What does **DNS** do?”
     → Resolves names to IP addresses.
   * “What does **ARP** do?”
     → Resolves IP addresses to MAC addresses on the local network.
   * “What does **DHCP** do?”
     → Automatically assigns IP config (IP, mask, gateway, DNS) to hosts.
   * “What is the main difference between **TCP** and **UDP**?”
     → TCP is connection-oriented and reliable; UDP is connectionless and no built-in reliability.

4. **Devices**

   * “How is a **hub** different from a **switch**?”
     → Hub floods everything, no MAC table; switch learns MAC addresses and forwards selectively.
   * “Why do we prefer switches over hubs?”
     → Better performance and less collision, more secure.

5. **Routing & Summarization**

   * “What is a **routing table**?”
     → A list of networks and how to reach them (next hop or interface).
   * “What is **route summarization** and why is it used?”
     → Combining multiple specific networks into a larger summarized route to reduce table size and improve efficiency.

6. **End-to-End Story**

   * “Walk me through, in 4–6 sentences, what happens when you go to `google.com`.”

*Let one brave soul try. Help them fill in: DHCP, DNS, ARP, default gateway, switch, router, internet, TCP/HTTP.*

### 8.2 Final Recap

**You:**
“Let’s recap the entire picture in human language:

* Your device gets its IP config from **DHCP**.
* When you type a URL, **DNS** translates the name to an IP.
* Your device checks: ‘Is this IP in my network? If not, send it to the **default gateway**.’
* To talk to the default gateway, it uses **ARP** to find the router’s MAC.
* Inside your LAN, **switches** move frames using MAC addresses (Layer 2).
* **Routers** move packets between networks using IP addresses and routing tables (Layer 3).
* **TCP or UDP** at Layer 4 handle the end-to-end transport (reliability, ports).
* At the top, **application protocols** like HTTP, DNS, SMTP live at Layer 7.

If you can tell that story and connect it to the OSI layers and devices, you’ve basically got the **core of computer networking** for a standard exam.”

**You (optional closing):**
“If you want to study after this, try to:

* Draw your home network.
* Label each device with its role (host, switch, router).
* Write down which protocols are involved when you open a web page.

If that feels easy, you’re in good shape for the final.”

---

## 🧠 Summary (in plain English):

You asked for a longer, speak-aloud script for a 2-hour computer networks tutoring session. I gave you a full, copy-paste-ready markdown document you can put in GitHub and literally read from. The script walks through:

* Intro + hook (what happens when you type `google.com`)
* Core concepts: hosts, MAC vs IP, networks, default gateway
* Devices: hub vs switch vs router, with analogies and exam-style distinctions
* OSI model with focus on Layers 1–4 (Physical, Data Link, Network, Transport)
* Deep, narrative walk-through of a packet’s journey: DHCP → DNS → ARP → switch → router → internet → server → back, touching TCP and HTTP/HTTPS
* Brief but clear section on router hierarchies and route summarization
* OSI Layers 5–7 and main application protocols (HTTP, DNS, DHCP, SMTP, etc.)
* A final section of rapid-fire exam questions and a clean conceptual recap.