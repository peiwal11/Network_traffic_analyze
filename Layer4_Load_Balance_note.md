## Load Balancing on Layer 4
Layer 4 load balancing involves distributing network traffic based on the information in the transport layer, such as TCP/UDP headers. It does not fully establish a TCP connection with both the client and the real server; instead, it inspects headers (like port and flags) and then relays the packets without delving into details like the sliding window.     

## Ways of Load Balancing   
1.	Client Visits Different IPs – DNS-Based Load Balancing
     
- __Issues__:  
   - Limitation of DNS Response: The number of IP addresses in a single DNS response is limited. To handle this, the DNS server can shuffle the response subset and always return a different set of IPs.   
   -	Static Order of DNS Choices: To avoid static order, the DNS response order can be shuffled.   
   -	DNS Caching: Traditional DNS queries are cached at multiple layers (OS, ISP, intermediate servers), causing delays in updates.
    
- __Solution__:   
   -	HTTP-DNS: Instead of traditional DNS queries, the app sends HTTPS requests to its own DNS server, bypassing intermediate caches and ensuring real-time DNS resolution.    

   
2.	Reverse Proxy Load Balancing   
- __Method__: Clients connect to a virtual IP (VIP), which then distributes traffic to different server IPs.  
- __Requirements for Load Balancers__:  
     -	Discovery of New Server Instances: The load balancer must detect and use new servers.  
     -	Scalability: Both the real servers and the load balancer itself must be scalable.  
     -	Performance: The load balancer must handle more traffic than individual servers while performing efficiently.
 
- __Nginx as a Layer 7 Load Balancer__: Addresses the above requirements but needs a Layer 4 load balancer for its own scalability.  
- __Layer 4 Load Balancer Requirements__:   
    -	High Availability: Achieved using ECMP (Equal-Cost Multipath) and VIP.
    -	Stable Long Connections: Ensuring connections remain stable even during topology changes.


## Relationship Between VIP and ECMP
-	VIP: Provides a single IP address for clients to connect to.   
-	ECMP (Equal-cost multipath): ECMP distributes traffic evenly across multiple paths with the same cost, calculated by routing protocols, to ensure efficient network utilization. The decision is based on the metric of the next hop.     
-	Hashing in ECMP and Load Balancing: A hash function (using source/destination IPs, ports, and protocol) determines which server to route to. For instance, with VIP bound to Server A and Server B, an even hash value might route to Server A and an odd value to Server B.
    
## Handling Long Connections
When adding or removing servers, the hash results can change, potentially disrupting long connections. To maintain connection stability:   
- Sticky ECMP: Ensures that existing connections continue to use the same server even when new servers are added. New connections will use the new servers.
  - Drawback: After old connections break, new connections from the old device may still go to the old server, not evenly distributing traffic to the new server   

-	Extra Proxy   
   - Director: each director maintains connection status from client to load balancer, yet state synchronize may delay, also director need to keep connection data   
   - Github GLB: keep primary and secondary LB instance, forward to primary LB instance, if couldn’t find corresponding connection status at such LB instance than forward to secondary LB instance.    
     Drawback:   
       -	One time could only increase one device  
       -	If second attempt still fail, them connections would be drop   
       -	Performance lower, cause add another relay


Yet above solutions only address add servers, it does not resolve server remove.    
The implementation of multiple LB to solve long connection problems needs to keep a state service outside to store connection status (this handles synchronize of connection status between each director). Also, in order to let the relay could come from every load balance instances, load balancer should keep not only client inward VIP but also real server inward VIP. This is indispensable to address the traffic could be handle from every load balancer. (this relay is facilitate by SNAT) (meaning the load balancer that handle inward traffic is not nesscary same as the load balancer that handle outward traffic)        

The relay of LB connecting client and real server could be done by using several ways.  
- Full NAT. (change SIP and DIP at the same time), in this way if need the info of client IP, use TOA (TCP Option Address).
-	DNAT (only change inward DIP , outward SIP – that is always remains client IP) with real server and LB under same subnet, real server source MAC IP set as LB, let LB than do the dnat. (LB and real server should under same subnet, ensure would only go through layer2 not layer3)   
Another way to do this is to use Virtual Switch, in that case on need to have LB and real server under same subnet   
-	DNAT+SNAT, these two are on different device    



However, the reason we deploy a load balancer is for security purposes, to prevent DDoS attacks, which typically target inbound traffic. Yet, the load balancer now spends most of its time on outbound traffic (sending out a great amount of data). To reduce the time spent on outbound traffic, we try to let data from our real server directly respond back to the client without going through the load balancer. We need to ensure the following requirements:
   1.	We can’t expose the real server’s IP.   
   2.	We must be able to address the client’s TCP request.
 
We are going to implement DSR (Direct Server Return).   
-	Configure VIP on Both LB and Real Server: Ensure both the Load Balancer and Real Server are configured with the VIP.   
-	Handle ARP Requests on the LB: Configure the Load Balancer to respond to ARP requests for the VIP and configure the Real Server to ignore ARP requests for the VIP. (Because ARP is used to answer the mapping between IP and MAC, when traffic comes in, it asks who owns the IP. In this circumstance, only the LB responds, so traffic goes to the LB.)   
-	Forward Packets Using MAC Addresses: The Load Balancer forwards packets to the Real Server using the Real Server’s MAC address.   
Since we are using MAC addresses to forward traffic from the LB to the real server, the LB and the real server need to be under the same subnet. To address this problem, we use "two layers of IP". We encapsulate the "client IP, LB VIP" inside a UDP packet to go through a third layer. (Now it is no longer limited to Layer 2.)    
Another problem with DSR is that it cannot obtain the complete status of the TCP connection. As the LB and the real server share the same VIP, and inbound traffic first passes through the LB before reaching the real server, if the real server sends an RST and the client responds and wants to set up a new connection, the LB does not know about it. Since the LB will receive the new connection packet before the real server, and does not have information about the new connection, the LB may mishandle the packets, possibly dropping them.

Consistent Hashing. The purpose of it is to minimize the number of connection reassignments when servers are added or removed. However, it cannot fully guarantee that existing connections will not be reassigned during topology changes. Thus, it can’t solve the long connection problem.



## Multiple kinds of LB handle relay

-  Nginx: Using Nginx as a relay is slow because it operates at the application layer (Layer 7) and relies on Linux syscalls, requiring complete TCP connections between the client, load balancer, and real server.   
-  LVS: We prefer using LVS (Linux Virtual Server) as it operates at the transport layer (Layer 4) within the kernel, avoiding user space and thus being faster.   
-  XDP (eBPF): XDP processes packets at the Network Interface Card (NIC) level using eBPF, bypassing more of the kernel network stack and not going up to Layer 4.   
-  DPDK: DPDK allows user space to directly interact with the NIC, bypassing the syscall and kernel, providing the fastest packet processing.   
(30 cores for 28Mpps means to achieve forwarding 28 million individual packets every second, need distributed across 30 individual processing units)   

## Arms (one-arm v.s two-arm)
How to determine it is one-arm or two-arm? The number of network interfaces typically determines how many "arms" a network device configuration has.   
-	One-arm: single network interface. For example: Router-on-a-Stick: A router uses one physical interface to connect to a switch and handles multiple VLANs through virtual sub-interfaces. This single interface handles both incoming and outgoing traffic for all VLANs.   
-	Two-arm: there are two separate physical network interfaces used. For example: a LB instance runs in WAN needs two network interface card, one for client WAN VIP another for LAN real server.  
