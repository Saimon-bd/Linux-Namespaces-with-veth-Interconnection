  
## Understanding Container Networking using Linux Network Namespaces to isolate the server.

#### Commands will be found [here](https://github.com/Saimon-bd/Linux-Namespaces-with-veth-Interconnection/blob/main/commnad-ns.sh)

#### What are Linux Namespaces?
**Linux Namespaces** `serves as an abstraction layer over operating system resources. Visualize a namespace as a container that encapsulates specific system resources, with each type of namespace representing a distinct box. Currently, there are `**7 (Seven)**` types of namespaces, namely` **`Cgroup`, `IPC (Inter-Process Communication)`, `Network`, `Mount`, `PID`, `User`, and `UTS (Unix Time-Sharing)`.**

#### What is Network Namespaces?

**Network namespace** `is a Linux kernel feature that allows us to isolate network environments through virtualization. For example, using network namespaces, you can create separate network interfaces and routing tables that are isolated from the rest of the system and operate independently. Network Namespace is a core component of Docker Networking.`

Network Namespaces, according to **`man 7 (Seven) network_namespaces`**:

**_network namespaces provide isolation of the system resources associated with networking: `Network devices,` `IPv4 and IPv6 protocol stacks,` `IP routing tables,` `Firewall rules,` `the /proc/net directory,` `the /sys/class/net directory,` `various files under /proc/sys/net,` `port numbers (sockets),` and so on._**

#### Virtual Interfaces and Bridges:
**_`Virtual interfaces`_** provide us with virtualized representations of physical `Network Interfaces`; and the **_`Bridge`_** gives us 
 the virtual equivalent of a `Switch`.

#### What are we going to cover?
- `We are going to create `**2 (Two) Network Namespaces**` (like two isolated servers), `**2 (Two) veth pairs**` (like two physical ethernet cables), and `**a bridge**` (for routing traffic between namespaces).`
- `Then we will configure the bridge so the two namespaces can communicate with each other.`
- `Then we will connect the bridge to the host and the internet`
- `At last, we will configure for incoming traffic(outside) to the namespace.`

## Let's start...

**_Step 0:_** To better understand, start by checking the basic network status on the host machine or root namespace. [ `I launch an ec2 instance(ubuntu) from AWS to simulate this hands-on. VM or even Normal Linux machines are also okay. `]
```bash
# List all the interfaces
sudo ip link

1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9001 qdisc fq_codel state UP mode DEFAULT group default qlen 1000
    link/ether 0a:1b:b1:bc:70:d0 brd ff:ff:ff:ff:ff:ff

# Find the routing table
sudo route -n

Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
0.0.0.0         172.31.0.1      0.0.0.0         UG    100    0        0 eth0
172.31.0.0      0.0.0.0         255.255.240.0   U     0      0        0 eth0
172.31.0.1      0.0.0.0         255.255.255.255 UH    100    0        0 eth0
```
**_Step 1.1:_** **Create two network namespace**
```bash
# Add two network namespaces using "ip netns" command
sudo ip netns add red
sudo ip netns add green

# List the created network namespaces
sudo ip netns list

red
green

# By convention, network namespace handles created by
# iproute2 live under `/var/run/netns`
sudo ls /var/run/netns/

red green
```
**_Step 1.2:_** **By default, network interfaces of created netns are down, even loop interfaces. make them up.**
```bash
sudo ip netns exec red ip link set lo up
sudo ip netns exec red ip link

1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00

sudo ip netns exec green ip link set lo up
sudo ip netns exec green ip link

1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
````

**_Step 2.1:_** **Create a bridge network on the host**
```bash
sudo ip link add br0 type bridge
# Up the created bridge and check whether it is created and in UP/UNKNOWN state
sudo ip link set br0 up
sudo ip link

3: br0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000
    link/ether 12:38:75:40:c0:17 brd ff:ff:ff:ff:ff:ff
```
**_Step 2.2:_** **Configure IP to the bridge network**
```bash
sudo ip addr add 192.168.1.1/24 dev br0
# check whether the ip is configured and also ping to ensure
sudo ip addr

3: br0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UNKNOWN group default qlen 1000
    link/ether 12:38:75:40:c0:17 brd ff:ff:ff:ff:ff:ff
    inet 192.168.1.1/24 scope global br0
       valid_lft forever preferred_lft forever
    inet6 fe80::1038:75ff:fe40:c017/64 scope link 
       valid_lft forever preferred_lft forever

ping -c 2 192.168.1.1

--- 192.168.1.1 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1026ms
rtt min/avg/max/mdev = 0.020/0.029/0.039/0.009 ms
```
**_Step 3.1:_** **Create two veth interfaces for two network netns, then attach to the bridge and netns**
```bash
# For red

# Creating a veth pair which have two ends identical veth0 and ceth0
sudo ip link add veth0 type veth peer name ceth0

# Connect veth0 end to the bridge br0
sudo ip link set veth0 master br0

# Up the veth0 
sudo ip link set veth0 up

# Connect ceth0 end to the netns red
sudo ip link set ceth0 netns red

# Up the ceth0 using 'exec' to run command inside netns
sudo ip netns exec red ip link set ceth0 up

# Check the link status 
sudo ip link

3: br0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP mode DEFAULT group default qlen 1000
    link/ether 9a:af:0d:89:8b:81 brd ff:ff:ff:ff:ff:ff
5: veth0@if4: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master br0 state UP mode DEFAULT group default qlen 1000
    link/ether 9a:af:0d:89:8b:81 brd ff:ff:ff:ff:ff:ff link-netns ns1

# check the link status inside red
sudo ip netns exec red ip link

1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
4: ceth0@if5: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP mode DEFAULT group default qlen 1000
    link/ether 3e:66:e5:b6:07:9a brd ff:ff:ff:ff:ff:ff link-netnsid 0


# For green; do the same as red

sudo ip link add veth1 type veth peer name ceth1
sudo ip link set veth1 master br0
sudo ip link set veth1 up
sudo ip link set ceth1 netns green
sudo ip netns exec green ip link set ceth1 up

sudo ip link 

3: br0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP mode DEFAULT group default qlen 1000
    link/ether 1a:f5:b2:8e:ca:a5 brd ff:ff:ff:ff:ff:ff
5: veth0@if4: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master br0 state UP mode DEFAULT group default qlen 1000
    link/ether 9a:af:0d:89:8b:81 brd ff:ff:ff:ff:ff:ff link-netns ns1
7: veth1@if6: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master br0 state UP mode DEFAULT group default qlen 1000
    link/ether 1a:f5:b2:8e:ca:a5 brd ff:ff:ff:ff:ff:ff link-netns ns2
    
sudo ip netns exec green ip link

1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
6: ceth1@if7: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP mode DEFAULT group default qlen 1000
    link/ether 3e:1e:48:de:47:07 brd ff:ff:ff:ff:ff:ff link-netnsid 0
```

**_Step 3.2:_** **Now we will add the IP address to the netns veth interfaces and update the routing table to establish communication with the bridge network it will also allow communication between two netns via the bridge;**
```bash
# For red
sudo ip netns exec red ip addr add 192.168.1.10/24 dev ceth0
sudo ip netns exec red ping -c 2 192.168.1.10
sudo ip netns exec red ip route
 
192.168.1.0/24 dev ceth0 proto kernel scope link src 192.168.1.10

# Check if you can reach the bridge interface
sudo ip netns exec red ping -c 2 192.168.1.1

--- 192.168.1.1 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1020ms
rtt min/avg/max/mdev = 0.046/0.050/0.054/0.004 ms

# For green
sudo ip netns exec green ip addr add 192.168.1.11/24 dev ceth1
sudo ip netns exec green ping -c 2 192.168.1.11
sudo ip netns exec green ip route 

192.168.1.0/24 dev ceth1 proto kernel scope link src 192.168.1.11

# Check if you can reach the bridge interface
sudo ip netns exec green ping -c 2 192.168.1.1

--- 192.168.1.1 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1020ms
rtt min/avg/max/mdev = 0.046/0.050/0.054/0.004 ms
```

**_Step 4:_** **Verify connectivity between two netns and it should work!**
```bash
# For red: 
# We can log in to netns environment using the below; It will be isolated from any other network
sudo nsenter --net=/var/run/netns/red

# ping to the green netns to verify the connectivity
ping -c 2 192.168.1.11

--- 192.168.1.11 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1019ms
rtt min/avg/max/mdev = 0.033/0.042/0.051/0.009 ms

# Exit from the red
exit

# For green
sudo nsenter --net=/var/run/netns/green

# Ping to the red netns to verify the connectivity
ping -c 2 192.168.1.10

--- 192.168.1.10 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1022ms
rtt min/avg/max/mdev = 0.041/0.044/0.048/0.003 ms

# Exit from the green
exit
```
#### Connectivity between two network namespaces via the bridge is completed.

![Project Diagram](https://github.com/Saimon-bd/Linux-Namespaces-with-veth-Interconnection/blob/main/namespaces.png)


**_Step 5.1:_** **Now it's time to connect to the internet. As we saw routing table from `ns1` doesn’t have a default gateway, it can’t reach any other machine from outside the `192.168.1.0/24` range.**
```bash
sudo ip netns exec red ping -c 2 8.8.8.8
ping: connect: Network is unreachable

# Check the route inside red
sudo ip netns exec red route -n

Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
192.168.1.0     0.0.0.0         255.255.255.0   U     0      0        0 ceth0
# As we can see, no route is defined to carry other traffic than 192.168.1.0/24

# We can fix this by adding a default route 
sudo ip netns exec red ip route add default via 192.168.1.1
sudo ip netns exec red route -n

Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
0.0.0.0         192.168.1.1     0.0.0.0         UG    0      0        0 ceth0
192.168.1.0     0.0.0.0         255.255.255.0   U     0      0        0 ceth0

# Do the same for green
sudo ip netns exec green ip route add default via 192.168.1.1
sudo ip netns exec green route -n

Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
0.0.0.0         192.168.1.1     0.0.0.0         UG    0      0        0 ceth1
192.168.1.0     0.0.0.0         255.255.255.0   U     0      0        0 ceth1

# Now first ping the host machine eth0
ip addr | grep eth0

2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9001 qdisc fq_codel state UP group default qlen 1000
    inet 172.31.13.55/20 brd 172.31.15.255 scope global dynamic eth0

# Ping from ns1 to host ip
sudo ip netns exec red ping 172.31.13.55
64 bytes from 172.31.13.55: icmp_seq=1 ttl=64 time=0.037 ms
64 bytes from 172.31.13.55: icmp_seq=2 ttl=64 time=0.036 ms

# we get the response from host machine eth0
```
**_Step 5.2:_** **Now let's see if ns1 can communicate to the internet, we can analysis traffic using tcpdump to see how a packet will travel. Open another terminal for catching traffic using tcpdump.**
```bash
# terminal-1
# now trying to ping 8.8.8.8 again
sudo ip netns exec red ping 8.8.8.8

# still unreachable
# terminal 2
# open tcpdump in eth0 to see the packet
sudo tcpdump -i eth0 icmp
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on eth0, link-type EN10MB (Ethernet), capture size 262144 bytes

# no packet captured, let's capture traffic for br0
sudo tcpdump -i br0 icmp

tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on br0, link-type EN10MB (Ethernet), capture size 262144 bytes
02:17:30.807072 IP ip-192-168-1-10.ap-south-1.compute.internal > dns.google: ICMP echo request, id 17506, seq 1, length 64
02:17:31.829317 IP ip-192-168-1-10.ap-south-1.compute.internal > dns.google: ICMP echo request, id 17506, seq 2, length 64

# We can see the traffic at br0 but we don't get a response from eth0.
# It's because of an IP forwarding issue
sudo cat /proc/sys/net/ipv4/ip_forward
0

# enabling ip forwarding by changing value 0 to 1
sudo sysctl -w net.ipv4.ip_forward=1
sudo cat /proc/sys/net/ipv4/ip_forward
1

# terminal-2
sudo tcpdump -i eth0 icmp
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on eth0, link-type EN10MB (Ethernet), capture size 262144 bytes

02:30:12.603895 IP ip-192-168-1-10.ap-south-1.compute.internal > dns.google: ICMP echo request, id 18103, seq 1, length 64
02:30:13.621367 IP ip-192-168-1-10.ap-south-1.compute.internal > dns.google: ICMP echo request, id 18103, seq 2, length 64

# As we can see how we are getting responses eth0
# but ping 8.8.8.8 still not working
# Although the network is now reachable, there’s no way that 
# We can have responses back - cause packets from external networks 
# can’t be sent directly to our `192.168.1.0/24` network.
```
**_Step 5.3:_** **To get around that, we can make use of NAT (network address translation) by placing an `iptables` rule in the `POSTROUTING` chain of the `nat` table.**
```bash
sudo iptables \
        -t nat \
        -A POSTROUTING \
        -s 192.168.1.0/24 ! -o br0 \
        -j MASQUERADE
# -t specifies the table to which the commands
# should be directed to. By default, it's `filter`.
# -A specifies that we're appending a rule to the
# chain then we tell the name after it;
# -s specifies a source address (with a mask in this case).
# -j specifies the target to jump to (what action to take).

# Now we're getting a response from Google DNS
sudo ip netns exec red ping -c 2 8.8.8.8

--- 8.8.8.8 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1002ms
rtt min/avg/max/mdev = 1.625/1.662/1.700/0.037 ms
```

**_Step: 6_** **Now let's open a service in one of the namespaces and try to get a response from outside**
```bash
sudo nsenter --net=/var/run/netns/netred
python3 -m http.server --bind 192.168.1.10 3000
```
`As I have an ec2 instance from AWS, it has an attached public IP. I will try to reach that IP with a specific port from outside.` 
```bash
telnet 65.2.35.192 5000
Trying 65.2.35.192...
telnet: Unable to connect to remote host: Connection refused
```
`As we can see we can't reach the destination. Because we didn't tell the Host machine where to put the incoming traffic. We have to NAT again, this time we will define the destination.`
```bash

sudo iptables \
        -t nat \
        -A PREROUTING \
        -d 172.31.13.55 \
        -p tcp -m tcp --dport 5000 \
        -j DNAT --to-destination 192.168.1.10:5000
# -p specifies a port type and --dport specifies the destination port
# -j specifies the target DNAT to jump to the destination IP with port.


# From my laptop
# Now I can connect the destination with the port.
# We successfully received traffic from the internet inside the container network

telnet 65.2.35.192 5000
Trying 65.2.35.192...
Connected to 65.2.35.192.
The escape character is '^]'.
```

#### Now the hands on completed; What we achieve:
- `The Host can send traffic to any application inside network namespaces` 
- `The Application inside network namespaces can communicate with host applications and other network namespace applications`
- `An application inside the network namespaces can connect to the internet`
- `An application inside the network namespaces can listen for requests from the outside internet`
- `Finally, we understand how Docker or some other container tool does networking under the hood. They automate the whole process for us when we give commands like "Docker run -p 5000:5000 frontend-app"`

#### Resources
- https://man7.org/linux/man-pages/man7/network_namespaces.7.html
- https://ops.tips/blog/using-network-namespaces-and-bridge-to-isolate-servers/
- https://github.com/dipanjal/DevOps/tree/main/NetNS_Ingress_Egress_Traffic
- https://man7.org/linux/man-pages/man8/iptables.8.html
