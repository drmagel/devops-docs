# Network fundamentals

## TCP/IP Protocol

### /IP Protocol Overview
The TCP/IP suite is a collection of communication rules that allow diverse devices to exchange data across networks.  
- **TCP (Transmission Control Protocol)**: A connection-oriented protocol that ensures reliable data delivery. It uses a three-way handshake (SYN, SYN-ACK, ACK) to establish a session, numbers packets for reordering, and retransmits lost data.
- **UDP (User Datagram Protocol)**: A connectionless, low-overhead protocol used for speed. It does not guarantee delivery or order, making it ideal for real-time traffic like VoIP, gaming, or DNS.  

### TCP/IP Model Layers
While the theoretical OSI model has 7 layers, the practical TCP/IP model is typically viewed in 4 or 5 layers. 

| Layer | Function | Key Protocols |
|-------|----------|--------------|
| 7. Application | User-facing services and data formatting. | HTTP, HTTPS, FTP, SMTP, DNS |
| 4. Transport | End-to-end communication and reliability. | TCP, UDP |
| 3. Internet (Network) | Routing and logical addressing (IP). | IPv4, IPv6, ICMP, ARP |
| 2. Data Link | Physical addressing (MAC) and framing. | Ethernet, Wi-Fi (802.11) |
| 1. Physical | Transmission of raw bits over hardware. | Cables, fiber optics, radio waves |

