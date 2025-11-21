
# **Acronyms & Meanings**

### **Core Telephony / Networking**

* **SIP** — Session Initiation Protocol
  *Signaling protocol for establishing phone calls.*

* **RTP** — Real-Time Transport Protocol
  *Carries the actual audio/video packets.*

* **SFU** — Selective Forwarding Unit
  *A media server that forwards audio/video streams to subscribed participants without mixing.*

* **STUN** — Session Traversal Utilities for NAT
  *Helps a WebRTC client discover its public IP/port.*

* **TURN** — Traversal Using Relays around NAT
  *Relays media when direct peer-to-peer connection fails (coturn is a common server).*

* **ICE** — Interactive Connectivity Establishment
  *The algorithm WebRTC uses to choose the best media path using STUN/TURN candidates.*

* **NAT** — Network Address Translation
  *Translates private IPs to public IPs; causes WebRTC connectivity issues.*

* **VPC** — Virtual Private Cloud
  *Isolated private network inside AWS or other cloud platforms.*

* **CIDR** — Classless Inter-Domain Routing
  *Defines IP address ranges (e.g., 10.0.0.0/16).*

* **NLB** — Network Load Balancer
  *Layer-4 load balancer often used for internal private traffic.*

* **DNS** — Domain Name System
  *Resolves domain names to IP addresses.*

---

# **WebRTC / Media**

* **WebRTC** — Web Real-Time Communications
  *Browser-based real-time audio/video data framework used by LiveKit.*

* **DTLS** — Datagram Transport Layer Security
  *Encrypts WebRTC media streams.*

* **SRTP** — Secure Real-Time Transport Protocol
  *Encrypted version of RTP used by WebRTC.*

* **Opus** — Audio codec optimized for speech.

* **PCMU / G.711** — Traditional telephony codec used by SIP endpoints.

---

# **Cloud & Infrastructure**

* **AWS** — Amazon Web Services
* **EC2** — Elastic Compute Cloud (virtual servers)
* **ECS** — Elastic Container Service (container orchestration)
* **Fargate** — Serverless compute for containers
* **IAM** — Identity and Access Management
* **AZ** — Availability Zone

---

# **Voice AI & Agents**

* **STT** — Speech-to-Text
  *Converts speech to text.*

* **TTS** — Text-to-Speech
  *Converts text to speech.*

* **ASR** — Automatic Speech Recognition
  *(Same general family as STT.)*

* **LLM** — Large Language Model
  *AI language generation engine.*

* **RAG** — Retrieval-Augmented Generation
  *LLM with document lookup.*

* **IVA** — Intelligent Virtual Agent
  *Voicebot / conversational AI system.*

---

# **LiveKit Ecosystem**

* **Egress** — Process of streaming/recording media out of LiveKit.
* **Ingress** — Process of bringing external RTMP/SIP/WebRTC streams into LiveKit.
* **SDK** — Software Development Kit (client libraries for LiveKit).

