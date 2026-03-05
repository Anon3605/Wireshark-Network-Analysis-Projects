Wireshark Network Reconnaissance Detection Analysis


Step 01: CApture File Properties
First we go to see the statistics of capturing file properties.

Here we see: 
First packet: 2026-03-05 18:51:02
Last packet: 2026-03-05 19:00:13
Elapsed: 00:09:10
Packets: 4819(captured) (100.0% Displayed)

Step 02: Protocol Hierarchy
Statistics -> Protocol Hierarchy
Here we see 2 main Protocols in IPv4 (4283/4819):
1. TCP (4059/4819)
2. ARP (597/4819)

Step 03: Conversations
Statistics -> Conversations

If we look in IPv4 tab, there are 4 internal IP addresses:
- 192.168.1.103
- 192.168.1.109
- 192.168.1.113
- 192.168.1.116
- 192.168.1.254

and 2 external IP addresses.

Most packets are from 192.168.1.113 to 192.168.1.216 and same amount of packets from both directions.

In TCP there are Long list of connections between 192.168.1.113 and 192.168.1.116. 

192.168.1.116 used port 39640,39641,39642,39896,39897,39898,57214 for single packets for 192.168.1.113's port 1.

Later we see using port 41761 and 41858 for multiple packets. 
For port 41761 and 41858, there are packets sended in 60B and replies in 54B constant. All the data were sent to 192.168.1.113 from port range 1-65389.

Here Comes the decission, 
Attacker: 192.168.1.116
Victim: 192.168.1.113

Step 04: Time settings:
Change the format for Real time Investigation and also include the delta time.
View -> Time Display Format -> UTC Format

Step 05: Filtering using Display Filters

Filter01: ip.src==192.168.1.116 && tcp.flags.syn==1
Result: 2008 packets

Filter02: (ip.src==192.168.1.113) && (ip.dst==192.168.1.116) && (tcp.flags.ack==1 && tcp.flags.syn==1)
Resullt: 0 packets

Filter03: (ip.src==192.168.1.113) && (ip.dst==192.168.1.116) && (tcp.flags.ack==1 && tcp.flags.reset==1)
Result: 2012 packets

Filter04: ip.src==192.168.1.116 && tcp.flags.ack==1
Result: 4 packets

Note: Here the server client role is attacker is the client and victim is the server. For a tcp connection to be established, the attacker (client) sends a SYN packet to the victim (server) and the victim (server) sends a SYN-ACK packet back to the attacker (client). The attacker (client) then sends an ACK packet to the victim (server) to complete the connection. Here we see that the attacker (client) is sending SYN packets to the victim (server) and the victim (server) is sending SYN-ACK packets back to the attacker (client). But the attacker (client) is not sending ACK packets to the victim (server) to complete the connection. This is a sign of a SYN flood attack.

As the victim (server) is sending RST-ACK packets, it means that the victim (server) is rejecting the connection attempts from the attacker (client) and all ports are closed. 


Filter06: eth.addr == 08:00:27:63:b0:05 && arp
From IP address of 192.168.1.116, we get his MAC address as 08:00:27:63:b0:05. For the ARP request statistics, we used the filter and got this result: 531 packets

ARP Timeline: 12:52:25-12:53:38
TCP Timeline: 12:52:33-12:55:08

COnclussion:
- ARP requests are sent from 192.168.1.116 to 192.168.1.254 (broadcast) to resolve the MAC address of 192.168.1.113.
- TCP connections are established from 192.168.1.116 to 192.168.1.113 on port 41761 and 41858.
- The victim (192.168.1.113) is sending RST-ACK packets to the attacker (192.168.1.116) to reject the connection attempts for closed ports.
- The attacker (192.168.1.116) is sending packets to the victim (192.168.1.113) on port 1, but the victim (192.168.1.113) is not sending packets back to the attacker (192.168.1.116). This is a sign of a port scan attack.
- Attackers not replying the last ACK packet means they are not completing the TCP handshake, which is a sign of a SYN flood attack.