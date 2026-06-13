# Linux Networking

Connecting machines together is fundamental to everything in DevOps.
Before you can build cloud infrastructure, deploy services, or set up CI/CD pipelines, you need to understand how Linux handles networking - how machines get addresses, how traffic is routed, and how different interfaces are managed.

This tutorial covers the core Linux networking primitives. The concepts here map directly to what you'll configure in AWS VPC.

## Network Interfaces

In the physical world, a machine connects to a network through a **Network Interface Card (NIC)** - a hardware component with a physical network port. In Linux, every networking device - physical or virtual - is represented as a **network interface**.

Use the `ip address` command (or `ip addr` for short) to list all interfaces:

```console
myuser@hostname:~$ ip addr
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
2: ens5: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9001 qdisc mq state UP
    link/ether 0a:5c:36:18:fe:82 brd ff:ff:ff:ff:ff:ff
    inet 10.1.1.1/24 metric 100 brd 10.1.1.255 scope global dynamic ens5
       valid_lft 3559sec preferred_lft 3559sec
```

Let's break down what we're seeing:

- **`lo`** - the **loopback** interface. A virtual interface used for communication between processes on the same machine. It always has IP `127.0.0.1`. When you run `curl localhost`, your machine resolves the name `localhost` to `127.0.0.1` via `/etc/hosts` - a simple text file that maps hostnames to IPs.
- **`ens5`** (or **`eth0`**) - the primary Ethernet interface, used to send and receive traffic with the outside world.

Notice that `ens5` is assigned the IP address `10.1.1.1/24`. Technically, IP addresses belong to **interfaces**, not to machines. A machine with multiple interfaces can have a different IP on each one.

Also notice the `link/ether 0a:5c:36:18:fe:82` line. This is the **MAC address** - a hardware-level identifier unique to each network interface. While IP addresses are logical and can change, MAC addresses are physical and fixed. When two machines communicate on the same local network, they address each other by MAC address at the hardware level. IP addresses come into play when traffic needs to travel beyond the local network - we'll cover that shortly.

## IP Addresses and Subnets

Every interface on a network must have a unique IP address. IP addresses are organized into **subnets** - logical groupings of addresses that share the same network prefix. All machines on the same subnet can communicate directly, without going through a router.

![][networking_subnets]

Consider the network above. Machines in the leftmost subnet all start with `10.1.1.xxx` - they share the same first 24 bits. We write this subnet as:

```
10.1.1.0/24
```

This is **CIDR notation** (Classless Inter-Domain Routing). The number after `/` is the **prefix length** - how many bits are fixed as the network portion. The remaining bits are available for host addresses.

| CIDR  | Fixed bits | Host bits | Total addresses | Usable hosts |
|-------|-----------|-----------|-----------------|--------------|
| `/24` | 24        | 8         | 256             | 254          |
| `/16` | 16        | 16        | 65,536          | 65,534       |
| `/8`  | 8         | 24        | 16,777,216      | 16,777,214   |

> [!NOTE]
> The first address in every subnet is the **network address** and the last is the **broadcast address** - both are reserved. So a `/24` gives you 254 usable host addresses, not 256.

The older equivalent notation is the **subnet mask**. `255.255.255.0` is the same as `/24` - the octets set to `255` mark the fixed (network) portion and `0` marks the variable (host) portion.

Use [cidr.xyz](https://cidr.xyz/) to explore CIDR notation interactively.

### Private IP ranges

Not all IP addresses can be used on the public internet. [RFC 1918](https://www.rfc-editor.org/rfc/rfc1918) reserves three ranges for private networks:

| Range            | Description                      |
|------------------|----------------------------------|
| `10.0.0.0/8`     | Large private networks           |
| `172.16.0.0/12`  | Medium private networks          |
| `192.168.0.0/16` | Home and small office networks   |

Traffic using these IPs is not routed on the public internet. You'll see these ranges used extensively in cloud VPCs and local networks.

## Route Table and Default Gateway

A machine can reach any host on its own subnet directly. But to reach a machine on a *different* subnet - or the internet - it needs a **router**.

The Linux kernel maintains a **routing table** that maps destinations to gateways. Inspect it with:

```console
myuser@hostname:~$ ip route
default via 10.1.1.4 dev ens5 proto dhcp metric 100
10.1.1.0/24 dev ens5 proto kernel scope link src 10.1.1.1
```

Or with the older `route -n` command (the `-n` flag shows numeric IPs instead of resolving hostnames, which is faster):

```console
myuser@hostname:~$ route -n
Kernel IP routing table
Destination     Gateway         Genmask           Flags Metric Ref    Use Iface
0.0.0.0         10.1.1.4        0.0.0.0           UG    100    0        0 ens5
10.1.1.0        0.0.0.0         255.255.255.0     U     100    0        0 ens5
```

Reading the table row by row:

- **Row 2** (`10.1.1.0/24`) - Traffic going to any machine in the local subnet stays local. Gateway is `0.0.0.0`, meaning "send directly, no router needed".
- **Row 1** (`0.0.0.0/0`) - Traffic going anywhere else should be forwarded to `10.1.1.4`. This is the **default gateway** - the router at the edge of your subnet that handles all traffic destined outside.

The `Iface` column tells the kernel which network interface to send the packet through. This is how the kernel knows that traffic for `10.1.1.0/24` goes out through `ens5` - that interface has an IP in that range.

### Longest prefix match

Both rows can match the same destination. For example, the IP `10.1.1.5` matches both `10.1.1.0/24` (prefix length 24) and `0.0.0.0/0` (prefix length 0).

Linux resolves this with the **longest prefix match** rule: the most specific route - the one with the longest prefix - wins. So traffic to `10.1.1.5` uses row 2 (prefix `/24` beats `/0`), and traffic to `8.8.8.8` uses row 1 (only `/0` matches).

## Testing Connectivity

Two essential tools for verifying network connectivity:

**`ping`** sends ICMP echo requests to check if a host is reachable:

```console
myuser@hostname:~$ ping 10.1.2.2
PING 10.1.2.2 56(84) bytes of data.
64 bytes from 10.1.2.2: icmp_seq=1 ttl=51 time=3.29 ms
64 bytes from 10.1.2.2: icmp_seq=2 ttl=51 time=3.27 ms
^C
```

Press `Ctrl+C` to stop. No reply means the host is unreachable or ICMP is blocked by a firewall.

**`traceroute`** shows the path packets take through the network - every router hop along the way:

```console
myuser@hostname:~$ traceroute 8.8.8.8
traceroute to 8.8.8.8 (8.8.8.8), 30 hops max
 1  10.1.1.4 (10.1.1.4)        0.5 ms
 2  203.0.113.1 (203.0.113.1)  2.1 ms
 3  8.8.8.8 (8.8.8.8)          3.4 ms
```

Each line is a hop. The first hop is always your default gateway.

## DHCP - How Machines Get Their IP Address

When a machine boots up and connects to a network, it doesn't know its IP address yet. The **Dynamic Host Configuration Protocol (DHCP)** automates this assignment.

![][networking_dhcp2]

The process follows four steps known as **DORA**:

1. **Discover** - The new machine broadcasts: "Is there a DHCP server out there?"
2. **Offer** - A DHCP server responds with an available IP address and lease duration.
3. **Request** - The client says: "I'd like that IP address."
4. **Acknowledge** - The server confirms: "It's yours."

This is why you see `proto dhcp` in `ip route` output - those routes were installed automatically when the machine booted and obtained its IP from a DHCP server.

# Exercises


### :pencil2: Traceroute within and across Availability Zones

In this exercise you'll use `traceroute` to observe how traffic behaves differently depending on whether two instances are in the same AZ or in different AZs.

1. Launch (or use a friend's) **two EC2 instances in the same subnet** (same AZ). SSH into instance A and run:
   ```bash
   traceroute <private-ip-of-instance-B>
   ```
   How many hops are there? What does that tell you about how the two instances are connected?

2. Now launch a **third instance in a different AZ** (but still in the same VPC). From instance A, run:
   ```bash
   traceroute <private-ip-of-instance-C>
   ```
   How many hops this time? Is there a difference compared to step 1? Why or why not?

3. From instance A, run `traceroute` to an external IP such as `8.8.8.8`. Compare the first hop to what you saw in steps 1 and 2. What is it, and why is it different?

4. Now compare **latency** - run `ping -c 5` to each of the three targets (instance B, instance C, `8.8.8.8`). What pattern do you notice in the round-trip times?


### :pencil2: Network namespaces

A **network namespace** is a Linux feature that gives a process its own completely isolated network stack - its own interfaces, route table, and firewall rules. Processes inside the namespace can't see or touch the network of processes outside it.

This is exactly how Docker isolates containers. When you run a container, Docker creates a new network namespace for it. The container thinks it's the only thing on its network.

Let's prove the isolation:

1. Check your current interfaces:
   ```bash
   ip addr
   ```
   Note down the interface names and IPs you see.

2. Create a new namespace called `netns-lab`:
   ```bash
   sudo ip netns add netns-lab
   ```

3. Run `ip addr` **inside** the namespace:
   ```bash
   sudo ip netns exec netns-lab ip addr
   ```
   What do you see? Compare it to step 1. Can the namespace see `ens5` (or `eth0`)? Can it see the host's IP?

4. From inside the namespace, try to ping Google:
   ```bash
   sudo ip netns exec netns-lab ping 8.8.8.8
   ```
   What happens? Run `sudo ip netns exec netns-lab ip route` to understand why.

5. Open two terminals side by side. In terminal 1, start a simple HTTP server on the host:
   ```bash
   python3 -m http.server 8888
   ```
   In terminal 2, try to reach it **from inside the namespace**:
   ```bash
   sudo ip netns exec netns-lab curl http://127.0.0.1:8888
   ```
   Does it work? Why not? Now try the same `curl` from the host (outside the namespace) - does that work?

   This is the core insight: the namespace has its own loopback. `127.0.0.1` inside the namespace is not the same `127.0.0.1` as on the host.

6. Clean up:
   ```bash
   sudo ip netns del netns-lab
   ```

> [!NOTE]
> We'll come back to network namespaces when we cover Docker. At that point you'll see how Docker connects a container's isolated namespace to the outside world using the exact techniques.

### :pencil2: Non-standard CIDR

When you create an AWS account, AWS automatically sets up a **default VPC** in every region - a ready-to-use virtual network so you can launch instances immediately without any networking setup. You'll notice it uses the CIDR `172.31.0.0/16`. Inside it, AWS automatically creates one default subnet per Availability Zone, each with the CIDR `/20` - for example `172.31.0.0/20`, `172.31.16.0/20`, `172.31.32.0/20`, and so on.

A `/20` is non-standard: the prefix doesn't end on an octet boundary - it cuts through the middle of the third octet.

Use [cidr.xyz](https://cidr.xyz/) with `172.31.0.0/20` to answer the following:

1. How many bits vary in this subnet?
2. How many total addresses does `172.31.0.0/20` contain? How many are usable on AWS (remember AWS reserves 5)?
3. In binary, what does the third octet of `172.31.0` look like? Which bits are fixed by the `/20` prefix and which are free?
4. The next default subnet is `172.31.16.0/20`. What changed in the third octet to get from `0` to `16`? Does that make sense given your answer above?
5. Is `172.31.15.255` inside `172.31.0.0/20`? What about `172.31.16.0`?
6. How many `/20` subnets can you carve out of the `172.31.0.0/16` VPC in total?

### :pencil2: EC2 with multiple network interfaces

In AWS, a network interface on an EC2 instance is called an **Elastic Network Interface (ENI)**. Every instance gets a primary ENI at launch, but you can attach additional ENIs - each with its own private IP, security group, and subnet. This maps directly to what you've seen with `ip addr` on Linux: one machine, multiple interfaces, each with its own IP.

A common use case is separating traffic types: one ENI for public-facing traffic, another for internal management or database access.

1. In the AWS console, go to **EC2 → Network Interfaces** and create two ENIs in the same subnet:
   - `<your-alias>-eni-0` 
   - `<your-alias>-eni-1`

2. Launch a `t3.micro` instance. At launch, attach both ENIs (under **Network settings → Add network interface**).

3. SSH into the instance and run `ip addr`. How many interfaces do you see (excluding `lo`)? What IP is assigned to each?

4. Run `ip route`. How many routes appear? Can you match each route to one of the ENIs?

5. Detach `<your-alias>-eni-1` from the instance while it is running. Run `ip addr` again - what changed? 

   > [!NOTE]
   > ENIs you create manually are not deleted when the instance terminates. The primary ENI (created automatically at launch) is deleted. This is why manually-created ENIs are useful for preserving a stable private IP across instance replacements.

[networking_subnets]: https://exit-zero-academy.github.io/DevOpsTheHardWayAssets/img/networking_subnets.png
[networking_dhcp2]: https://exit-zero-academy.github.io/DevOpsTheHardWayAssets/img/networking_dhcp2.png

