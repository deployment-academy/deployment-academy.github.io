---
title: "Linux Networking Refresh: Network Namespaces"
description: "A hands-on refresh on Linux network namespaces. Starting from two namespaces that can't even ping themselves, we connect them with a veth pair, replace it with a bridge, give the host a leg on that network, and work outward through routing, forwarding, NAT, and port forwarding — everything by hand with ip and iptables, ending at the rules Docker generates for you."
date: 2026-09-12
lastmod: 2026-09-12
draft: false
sidebar: "right"
widgets:
  - "ddg-search"
  - "recent"
  - "social"
categories:
  - "Key Concepts"
tags:
  - "networking"
  - "linux"
  - "network namespaces"
  - "iptables"
  - "nat"
  - "bridge"
  - "containers"
  - "lima"
  - "tcpdump"
---

In this tutorial we are going to do a refresh on Linux network namespaces. We will start with two namespaces that can't even ping themselves, connect them with a virtual cable, replace that cable with a bridge, give the host a leg on that bridge, and then work our way outward — routing, forwarding, NAT, route specificity, and finally port forwarding back in. Everything is done by hand with `ip` and `iptables`, so that nothing is hidden behind tooling.

To simulate an environment we will use a Linux machine via [Lima](https://lima-vm.io/) and the `limactl` CLI. Namespaces don't need a complicated topology — one VM is enough, because the whole point is that we're going to build a little network *inside* it.

<!--more-->

To follow along, you should be able to use Lima from any OS you might be using. See the [installation guide](https://lima-vm.io/docs/installation/).

This tutorial was created with Lima 2.2.0.

```bash
limactl --version
limactl version 2.2.0
```

## The setup

Note that our goal here is not to dive into Lima but to use it as supporting infrastructure for the networking concepts. For this reason, I will give most of the `limactl` commands without much explanation, unless they add to our main goal. To learn more about Lima, check its [documentation](https://lima-vm.io/docs/) or the `limactl` help.

Create one isolated network and one VM on it:

```bash
limactl network create nsnet --gateway 10.40.1.1/24
limactl create --name=nshost --network=lima:nsnet template:ubuntu -y
limactl start nshost
```

You can check the network with `limactl network list`. I'm omitting the output, but you should see some default networks (`bridge`, `shared`, etc.) and the one we just created: `nsnet`, listed with `MODE: user-v2` — a software-defined network.

One thing worth knowing about Lima before we go further: passing `--network` doesn't *add* an interface next to a default one — it *replaces* the default NIC. An instance created with no `--network` gets a single `eth0` on Lima's built-in default network; an instance created with one `--network=lima:<name>` still gets a single `eth0`, but it's on `<name>`'s subnet instead. `limactl shell` keeps working either way, since SSH access doesn't depend on that default network. So `nshost` will have exactly one interface, `eth0`, on `10.40.1.0/24`.

Let's see what address it got:

```bash
limactl shell nshost -- ip -4 addr show eth0
```

**Check this carefully rather than assuming, and write down what you see.** Lima's usernet DHCP hands out the **gateway address itself** (`.1`) to the first instance that requests a lease on a network, which surprises people — there's a good chance `nshost` is sitting on `10.40.1.1`. It might equally be `10.40.1.2` or something else in the range. Either is fine; we just need to know which.

Throughout this tutorial I'll call that value **the host's eth0 address**, and I'll write it as `10.40.1.1` in the concrete examples — the value Lima handed out on my run — so the commands stay readable. If yours differs, substitute yours. The good news is that we only actually *need* it in one section (port forwarding), so it's not a landmine everywhere.

Now, two addresses that matter and are easy to mix up, so let's pin them down before we touch anything:

- `10.40.1.0/24` is the **Lima network** — the "outside world" as far as this VM is concerned. The host's `eth0` lives here, and so does the network's gateway, `10.40.1.1`.
- `10.20.1.0/24` is the network we are about to invent inside the VM for our namespaces. It does not exist yet. Nothing outside this VM will ever know it exists. That last part turns out to matter a great deal later on.

Get a shell on the VM and install what we need:

```bash
limactl shell nshost

sudo apt-get update
sudo apt-get install -y iproute2 net-tools
```

`iproute2` (the `ip` command) is already there on Ubuntu; `net-tools` gives us `arp` and `route`, which are the legacy commands but still the ones most people reach for and still what you'll find in half the documentation on the internet. I'll use them where the older form is more familiar and show the modern `ip` equivalent alongside.

Everything we do from here is not persistent. It's all live kernel state manipulated with `ip` and `iptables`, which is exactly what makes it good for learning. I'll flag the persistent equivalents as we go and again at the end.

One last piece of shorthand, since we'll use it constantly: `ip -n ns0 <something>` is exactly equivalent to `ip netns exec ns0 ip <something>`. It's the same command, just less typing. For anything that isn't `ip` itself — `ping`, `arp`, `route`, `tcpdump`, a web server — you need the full `ip netns exec ns0 <command>` form.

## Creating namespaces

```bash
sudo ip netns add ns0
sudo ip netns add ns1

ip netns list
```

You should see `ns0` and `ns1` listed.

So what did we just create? A **network namespace** is a completely independent copy of the kernel's networking stack. Its own set of network interfaces. Its own routing table. Its own ARP table. Its own iptables rules and conntrack state. Its own port number space, so two namespaces can both have something listening on port 8080 without any conflict at all.

A process running inside a namespace sees only that stack. It cannot see the host's interfaces, cannot use the host's routes, and cannot reach anything the namespace isn't wired up to reach. The isolation is the default, and connectivity is the thing you have to build.

Let's confirm how complete that isolation is:

```bash
sudo ip netns exec ns0 ip link show
```

You get exactly one interface: `lo`, and it's in state `DOWN`. That's it. No `eth0`, no route to anywhere, nothing, nada.

It's worth dwelling on how empty that is. This namespace can't reach the internet, can't reach the host, can't reach `ns1` — and can't even reach *itself*:

```bash
sudo ip netns exec ns0 ping -c2 127.0.0.1
```

That fails with `connect: Network is unreachable`. The loopback interface exists but is down, so there's no connected route for `127.0.0.0/8` and the kernel has nowhere to send the packet. Bring it up:

```bash
sudo ip -n ns0 link set lo up
sudo ip -n ns1 link set lo up

sudo ip netns exec ns0 ping -c2 127.0.0.1
```

Now it replies. A fresh namespace is a machine with the network cable unplugged *and* the loopback switched off — you have to build everything.

## Connecting two namespaces with a veth pair

The tool for connecting namespaces is the **veth pair**: two virtual interfaces created together, wired back-to-back. Anything that goes into one end comes out the other. Think of it as a virtual Ethernet cable with a NIC soldered on each end. You always create both ends at once, and they always exist as a pair.

```bash
sudo ip link add veth-ns0 type veth peer name veth-ns1
```

At this point both ends are in the host's namespace, which does us no good — it's a cable with both plugs in the same socket. Move one end into each namespace:

```bash
sudo ip link set veth-ns0 netns ns0
sudo ip link set veth-ns1 netns ns1
```

Check that they landed:

```bash
sudo ip -n ns0 link show
sudo ip -n ns1 link show
ip link show | grep veth
```

`ns0` now has `lo` and `veth-ns0`; `ns1` has `lo` and `veth-ns1`; and the host has neither, because moving an interface into a namespace really does *move* it. Notice also that each end reports its peer index — something like `veth-ns0@if3` — so you can always find the other side of a cable.

Now give them addresses. And here is the first thing worth being careful about:

```bash
sudo ip -n ns0 addr add 10.20.1.1/24 dev veth-ns0
sudo ip -n ns1 addr add 10.20.1.2/24 dev veth-ns1

# to verify the IPs
sudo ip -n ns0 -4 addr show
sudo ip -n ns1 -4 addr show
```

Note the `/24`. It matters more than it looks. If you omit the prefix length, `ip addr add` doesn't guess "probably a /24" — it defaults to **/32**, a network containing exactly one address. The kernel would then create a connected route for `10.20.1.1/32` only, `ns0` would have no idea that `10.20.1.2` was on the same wire, and the ping would fail with `Network is unreachable` even though the cable is perfectly fine.

If you want to see that for yourself rather than take my word for it, add the addresses without the prefix, try the ping, then compare `ip -n ns0 route show` against what you get with `/24`. With `/32` there's no subnet route at all. **Always write the prefix length**; the failure it causes looks exactly like a wiring problem and isn't one.

Now bring the interfaces up:

```bash
sudo ip -n ns0 link set veth-ns0 up

# optionally you can check veth-ns0
sudo ip -n ns0 link show

sudo ip -n ns1 link set veth-ns1 up
```

A veth pair only carries traffic when **both** ends are up. One end down and the other reports `NO-CARRIER` — it's a cable with nothing plugged into the far side.

Test it:

```bash
sudo ip netns exec ns0 ping -c3 10.20.1.2
```

Replies. Two isolated namespaces, one virtual cable, connectivity.

Have a look at what the ping left behind:

```bash
sudo ip netns exec ns0 arp
# the modern equivalent
sudo ip -n ns0 neigh show
```

You'll see an entry for `10.20.1.2` with the MAC address of `veth-ns1` and state `REACHABLE`. That's ARP doing exactly what it does on physical Ethernet: before `ns0` could send an IP packet to `10.20.1.2`, it had to find out which hardware address to put in the frame, so it broadcast an ARP request, `ns1` answered, and the result got cached. Nothing about the virtual interfaces changed the protocol. Compare the MAC against `ip -n ns1 link show veth-ns1` and you'll see they match.

While you're here, look at the routing table too:

```bash
sudo ip -n ns0 route show
```

One line: `10.20.1.0/24 dev veth-ns0 proto kernel scope link src 10.20.1.1`. That's the **connected route**, created automatically by the `ip addr add` with the `/24`. It says "this whole subnet is directly reachable out this interface, no gateway needed." It is the only route `ns0` has, which is why the next few sections are going to be about the things it can't reach.

## Scaling up with a Linux bridge

A veth pair connects exactly two things. That's fine for two namespaces and hopeless beyond that: for N namespaces to all reach each other with direct cables you need N(N-1)/2 pairs, and each namespace needs N-1 interfaces with N-1 addresses. Three namespaces is 3 cables, ten is 45, fifty is 1,225. This is exactly why physical networks stopped being point-to-point cables and got switches.

The Linux equivalent of a switch is a **bridge**. It's a virtual layer-2 device: you attach interfaces to it (they become its *ports*), it learns which MAC address it saw on which port, and it forwards frames accordingly — flooding to all ports when it hasn't learned the destination yet. Same behaviour as a physical switch, implemented in the kernel. Now each namespace needs exactly one veth pair, with one end in the namespace and the other plugged into the bridge: N pairs instead of N(N-1)/2.

Let's start by tearing down the direct cable:

```bash
sudo ip -n ns0 link del veth-ns0
```

Only one command, and both ends are gone — check with `ip -n ns1 link show` and `veth-ns1` is no longer there. Deleting either end of a veth pair automatically deletes the other; a cable can't have one end.

Now create the bridge:

```bash
sudo ip link add br-ns type bridge
sudo ip link set dev br-ns up
```

Now build the two new pairs, one per namespace, with a naming convention that says where each end goes:

```bash
# ns0's cable: one end in the namespace, one end on the bridge
sudo ip link add veth-ns0 type veth peer name veth-ns0-bridge
sudo ip link set veth-ns0 netns ns0
sudo ip link set veth-ns0-bridge master br-ns

# ns1's cable, same shape
sudo ip link add veth-ns1 type veth peer name veth-ns1-bridge
sudo ip link set veth-ns1 netns ns1
sudo ip link set veth-ns1-bridge master br-ns
```

`master br-ns` is what attaches an interface to the bridge — it makes that interface a port on the switch.

Now the interfaces. There are four of them, and all four need to be up. Before proceeding you can check the interfaces with `link show` in the two namespaces and in the host.

```bash
# the namespaces
sudo ip -n ns0 link show
sudo ip -n ns1 link show
# and the host
sudo ip link show
```

Set the interfaces up.

```bash
# the namespace ends
sudo ip -n ns0 addr add 10.20.1.1/24 dev veth-ns0
sudo ip -n ns1 addr add 10.20.1.2/24 dev veth-ns1
sudo ip -n ns0 link set veth-ns0 up
sudo ip -n ns1 link set veth-ns1 up

# the bridge ends — easy to forget, and nothing works without them
sudo ip link set veth-ns0-bridge up
sudo ip link set veth-ns1-bridge up
```

As we saw with the direct link both ends of the veth pair need to be up or the other reports `NO-CARRIER`. Attaching an interface to a bridge with `master` does not bring it up, and a bridge port that is administratively down forwards nothing. The symptom is a ping that fails silently with no error at all — no `Network is unreachable`, because the route exists and looks fine; just no replies. If you'd like to see it, leave those two commands out, watch the ping fail, then run them and watch it start working mid-ping.

We had to re-add the addresses because deleting `veth-ns0` deleted the interface, and an address belongs to an interface, not to a namespace. The namespaces themselves survived — they still exist, still have their `lo` up — but everything attached to the deleted interface went with it.

Check the bridge:

```bash
bridge link show
ip -d link show br-ns
```

You should see both `veth-ns0-bridge` and `veth-ns1-bridge` listed with `master br-ns` and state `forwarding`.

Now test:

```bash
sudo ip netns exec ns0 ping -c3 10.20.1.2
```

Same addresses, same result, completely different path. The packet leaves `ns0` on `veth-ns0`, pops out at `veth-ns0-bridge`, and the bridge forwards it to `veth-ns1-bridge` based on the destination MAC, from where it emerges inside `ns1`. Have a look at what the bridge learned:

```bash
bridge fdb show br br-ns
```

That's the **forwarding database** — MAC addresses mapped to ports, populated by observing traffic, exactly like a physical switch's CAM table. Adding a third namespace now means one more veth pair and one more bridge port; nothing about `ns0` or `ns1` changes at all. That's the scaling property we came for.

## Giving the host a leg on the namespace network

Right now the two namespaces can talk to each other and to nobody else — including the host they're running on. The host created the bridge, but it has no address on `10.20.1.0/24`, so it has no route to it and no way in.

The fix is a single command. A bridge in Linux is not only a switch, it's also an interface in its own right, and you can give it an address:

```bash
sudo ip addr add 10.20.1.5/24 dev br-ns
```

That gives the host itself an address on the namespace network — the equivalent of plugging the host into its own switch. It also creates a connected route in the host's routing table:

```bash
ip route show | grep 10.20.1
```

You should see `10.20.1.0/24 dev br-ns proto kernel scope link src 10.20.1.5`.

Test in both directions:

```bash
# from the namespaces to the host
sudo ip netns exec ns0 ping -c2 10.20.1.5
sudo ip netns exec ns1 ping -c2 10.20.1.5

# from the host to the namespaces
ping -c2 10.20.1.1
ping -c2 10.20.1.2
```

All four work, and no routes had to be added anywhere — `10.20.1.5` is on `10.20.1.0/24`, which every party already has a connected route for.

Here's the thing to hold onto, because the rest of the tutorial depends on it: the host now has a foot in **two** networks. `br-ns` at `10.20.1.5` on the namespace side, and `eth0` on `10.40.1.0/24` on the Lima side. A machine with a leg in two networks is precisely what a router is. We just haven't asked it to do the job yet. To learn more about routing check the [previous tutorial in this series](https://deployment.properties/posts/networking/linux-routing-refresh/).

## Reaching beyond the host

Let's try to reach something outside the namespace network. We'll use `8.8.8.8` — Google's public DNS resolver, which needs no second VM to exist and is definitively not on any network we've built.

Be clear about what that address is standing in for. It's simply "some address out on the wider network that the host can reach but the namespaces have never heard of". In a real setup this would be another server, a database, an API on your corporate network, or a machine in another VPC. Nothing in what follows depends on it being a DNS server; it's just a convenient reachable address that is definitively *not* on `10.20.1.0/24`.


Confirm the host can reach it, so we know the target isn't the problem:

```bash
ping -c2 8.8.8.8
```

Fine. Now from a namespace:

```bash
sudo ip netns exec ns1 ping -c2 8.8.8.8
```

```
connect: Network is unreachable
```

Before the explanation, look at why:

```bash
sudo ip netns exec ns1 route
# modern equivalent
sudo ip -n ns1 route show
```

One route, for `10.20.1.0/24`. The destination `8.8.8.8` matches nothing, there's no default route to catch it, and so the kernel refuses the packet before it's even built. `Network is unreachable` is the routing table telling you it has no idea where to send this — it isn't a timeout, it isn't a firewall, nothing left the machine at all.

The host is the obvious candidate for a gateway: it's on our network at `10.20.1.5` and it's also on `10.40.1.0/24`, with a path onward from there. Since we want `ns1` to reach *anything* out there, not one specific subnet, the route to add is the **default route** — the `0.0.0.0/0` prefix, which matches any destination not matched by something more specific:

```bash
sudo ip netns exec ns1 ip route add default via 10.20.1.5
```

Read it as: "anything you don't otherwise know what to do with, hand to `10.20.1.5`." That gateway has to be an address `ns1` can already reach directly, and it is — it's on the connected `10.20.1.0/24` route.

Try again:

```bash
sudo ip netns exec ns1 ping -c2 8.8.8.8
```

Different failure. No more `Network is unreachable`; now the pings just go out and nothing comes back. `100% packet loss`.

This is the point where `ping` has told us everything it can. It only knows "no reply came back", and that's the same output whether the request never got anywhere near the destination or arrived perfectly and the reply got lost coming home. Those are very different problems with the same symptom. To tell them apart we have to stop asking the endpoints and go watch the traffic.

Open a second session on the VM (`limactl shell nshost` again) and start a capture:

```bash
sudo tcpdump -i any -n icmp
```

Then re-run the ping from the first session, and read what you get. There are actually two things worth working out here, and I'd rather you found them than took my word for it: **does the request leave `br-ns` at all, and does anything ever appear on `eth0`?**

### What the capture shows, and why

With the capture running and the ping going, you should see the echo requests arriving on `br-ns` from `10.20.1.2` — and then nothing. They don't appear on `eth0`. The host received them and dropped them on the floor.

That's the first problem, and it's not about namespaces at all. A Linux host, by default, only handles traffic addressed **to itself**. Accepting a packet on one interface that's destined for a different machine and sending it out another interface is *forwarding*, and it's off unless you explicitly turn this host into a router:

```bash
sysctl net.ipv4.ip_forward
# or, equivalently
cat /proc/sys/net/ipv4/ip_forward
```

Very likely `0`. (Not always — installing Docker, among other things, turns it on. If yours is already `1`, you'll have watched the request reach `eth0` in the capture instead, and you can skip straight to the next part.)

Turn it on:

```bash
sudo sysctl -w net.ipv4.ip_forward=1
```

Now ping again with the capture still running:

```bash
sudo ip netns exec ns1 ping -c2 8.8.8.8
```

Still no replies — but the capture looks different now. You should see the echo request twice: once arriving on `br-ns`, and once leaving on `eth0`, still with source `10.20.1.2`. The packet is now making it out of the VM. Forwarding was **necessary but not sufficient**, and the capture is what tells us so.

So the request is going out. Why is nothing coming back? Look at that source address in the capture and think about what the destination has to do with it.

The request arrives at its destination claiming to come from `10.20.1.2`, and the reply has to find its way back to that address. It can't. `10.20.1.0/24` exists purely inside `nshost`; we invented it a few sections ago with `ip link add` and `ip addr add`, and we told precisely nobody. Worse, it's [RFC 1918](https://datatracker.ietf.org/doc/html/rfc1918) private space — every network in the world has its own idea of what `10.x` means, so no router on the public internet will carry it. The reply is dropped somewhere out there on the return path, and no amount of configuration on our side of the wire fixes that.

The same thing happens in miniature on a private network, for a milder reason: another machine on `10.40.1.0/24` would drop the reply simply because it has no route to `10.20.1.0/24`, which is a gap you *could* fill in. That gives us, broadly, two ways out. One is to make the outside world aware of `10.20.1.0/24` — add routes on the other machines, or run a routing protocol. That's the honest solution and it's what you'd do between subnets you control (it's how a normal router works), though it's not available to us against a destination on the public internet. The other is to stop showing the outside world those addresses at all. That's NAT.

### NAT and masquerading

```bash
sudo iptables -t nat -A POSTROUTING -s 10.20.1.0/24 -j MASQUERADE
```

Piece by piece: `-t nat` selects the NAT table. `POSTROUTING` is the chain traversed just before a packet leaves the box, after the routing decision has been made — the last possible moment to rewrite it. `-s 10.20.1.0/24` matches only traffic sourced from our namespace network. And `MASQUERADE` rewrites the source address of those packets to the address of the interface they're going out of.

So the destination never sees `10.20.1.2` at all. It sees a packet from the host's `eth0` address, which is an ordinary routable address that it knows perfectly well how to reply to. The reply comes back to the host, and the kernel's **connection tracking** — which recorded the original source when it rewrote the packet — recognises the reply, reverses the translation, puts `10.20.1.2` back as the destination, and forwards it down to `ns1`. `ns1` sees a normal reply from `8.8.8.8` and knows nothing about any of it.

`MASQUERADE` is a special case of `SNAT`. With plain `SNAT` you specify the new source address explicitly; `MASQUERADE` looks up the outgoing interface's current address for you. That's slightly more expensive per packet, and it's what you use when the address isn't known in advance or can change — a DHCP-assigned interface, a dial-up link, or exactly our situation, where we haven't hard-coded what Lima's DHCP handed out.

Now ping again, with the capture still running:

```bash
sudo ip netns exec ns1 ping -c2 8.8.8.8
```

Replies. And the capture is the interesting part — you should see something like this (with your own eth0 address in place of `10.40.1.1`):

```
IP 10.20.1.2 > 8.8.8.8: ICMP echo request, id 12, seq 1, length 64
IP 10.40.1.1 > 8.8.8.8: ICMP echo request, id 12, seq 1, length 64
IP 8.8.8.8 > 10.40.1.1: ICMP echo reply, id 12, seq 1, length 64
IP 8.8.8.8 > 10.20.1.2: ICMP echo reply, id 12, seq 1, length 64
```

That's the same packet twice on the way out and the same packet twice on the way back, captured on both interfaces, with the rewrite visible in between. The request goes in as `10.20.1.2` and comes out as the host's eth0 address; the reply arrives for the host and gets turned back into something for `10.20.1.2`. Seeing the translation happen in a capture is worth more than any amount of prose about it.

#### Optional: understand in details what's going on

You can inspect the rule and, more usefully, its counters:

```bash
sudo iptables -t nat -L POSTROUTING -n -v
```

```
Chain POSTROUTING (policy ACCEPT 1 packets, 216 bytes)
 pkts bytes target     prot opt in     out     source               destination
    1    84 MASQUERADE  all  --  *      *       10.20.1.0/24         0.0.0.0/0
```

Those `pkts` and `bytes` columns are the first thing to check when a NAT rule "isn't working" — if the counter is stuck at zero, the traffic isn't matching the rule at all and the problem is upstream of the NAT, not in it. `-n` keeps it from doing reverse DNS on every address (which makes the command hang on a broken network, delightfully), and `-v` is what gives you the counters in the first place.

Now compare that counter against what you just watched in `tcpdump`. We sent `-c2`, and the capture showed two echo requests going out — but the rule counter says **1**. That isn't an error, and it's worth understanding because it trips people up constantly.

**The NAT table only sees the first packet of a connection.** When that packet traverses `POSTROUTING` and gets masqueraded, conntrack records the translation. Every subsequent packet belonging to the same flow — the rest of the requests, and all the replies coming back — is recognised by conntrack and translated directly, without ever traversing the NAT chains again. NAT rules make the *decision* once per connection; conntrack does the *work* for the remaining packets.

(The `84` bytes confirms it: one ICMP echo request is 84 bytes on the wire — 64 bytes of payload, 8 of ICMP header, 20 of IP header. That's a single packet, not two.)

So the counter answers "did any traffic match this rule and start a translated connection?" — not "how many packets were rewritten?". If you want per-packet visibility, `tcpdump` is the tool, which is exactly why we watched the translation there rather than inferring it from these numbers.

Try it if you want to be sure: `sudo iptables -t nat -Z POSTROUTING` zeroes the counters, then run `ping -c4` from `ns1` and check again. Four packets out, four replies back in the capture, and the counter still reads `1`. (Inspecting the conntrack table itself needs the `conntrack` package — `sudo apt-get install -y conntrack`, then `sudo conntrack -L` — which we won't install here, since the counter already tells the story.)

## Route specificity

`ns1` can reach the outside world now. `ns0` still can't — we only ever gave the default route to `ns1`. Fix that, and then look at how the routing table makes its decisions:

```bash
sudo ip netns exec ns0 ip route add default via 10.20.1.5
sudo ip netns exec ns0 ping -c3 8.8.8.8
```

That works, and the path is worth spelling out because every piece we built is being used at once: `ns0` -> default route -> `br-ns` -> host forwards -> `eth0` -> MASQUERADE rewrites the source -> Lima's network -> out.

Now to routing decisions. Add a more specific route alongside the default, purely to see how the two interact:

```bash
sudo ip netns exec ns1 ip route add 10.40.1.0/24 via 10.20.1.5

sudo ip -n ns1 route show
```

```
default via 10.20.1.5 dev veth-ns1
10.20.1.0/24 dev veth-ns1 proto kernel scope link src 10.20.1.2
10.40.1.0/24 via 10.20.1.5 dev veth-ns1
```

Three routes, and routing always prefers the **most specific match** — the longest prefix that contains the destination, regardless of what order the routes appear in. So `10.20.1.x` matches the `/24` connected route and goes out directly with no gateway; `10.40.1.x` matches the `/24` we just added; and everything else, `8.8.8.8` included, falls through to the `/0` default. That `10.40.1.0/24` route is entirely redundant here, since the default already sends those packets to the same gateway — it's in the table only to make the precedence visible. Remove it again if you like:

```bash
sudo ip netns exec ns1 ip route del 10.40.1.0/24 via 10.20.1.5
```

The default route is what makes a host reachable-outward at all, and it's the one line you'll add in almost every real namespace or container setup. Everything more specific exists to carve exceptions out of it.

Note that we pinged an **address**, not a name. Name resolution inside a namespace is a separate concern with its own gotcha: a namespace does not automatically get its own `/etc/resolv.conf`, and by default `ip netns exec` gives processes the host's one. To override it per namespace, create `/etc/netns/<name>/resolv.conf` on the host and `ip netns exec` will bind-mount it over `/etc/resolv.conf` for processes in that namespace:

```bash
sudo mkdir -p /etc/netns/ns1
echo "nameserver 8.8.8.8" | sudo tee /etc/netns/ns1/resolv.conf
sudo ip netns exec ns1 ping -c2 google.com
```

That's as far as we'll go on DNS here — it's a whole topic of its own and out of scope for this one.

## Port forwarding into a namespace with DNAT

MASQUERADE solved outbound. Now the other direction: something outside wants to reach a service running *inside* a namespace. The namespace's address is unroutable from outside, so the only way in is for the host to accept the connection on an address the outside world can reach and rewrite the destination. That's **DNAT** — destination NAT — and in this form it's what everyone calls port forwarding.

First, put something inside `ns1` worth reaching:

```bash
# in a second session on the VM, leave this running
sudo ip netns exec ns1 python3 -m http.server 8080 --bind 0.0.0.0
```

Before testing any NAT, confirm the server itself works. The host has a direct route to `ns1` over the bridge, so it can reach it without any translation involved:

```bash
curl -s -o /dev/null -w '%{http_code}\n' http://10.20.1.2:8080
```

`200`. Good — the service is up, so anything that fails from here on is the NAT and not the server.

Now the test that matters. We want something to reach that service *without* knowing `10.20.1.2` exists, by connecting to an address the outside world can actually route to — the host's `eth0`. We'll send it from `ns0`, which is on the far side of the bridge, so its packets arrive at the host as genuine inbound traffic:

```bash
# 10.40.1.1 is the host's eth0 address — check yours with `ip -4 addr show eth0`
sudo ip netns exec ns0 curl -s -m3 -o /dev/null -w '%{http_code}\n' http://10.40.1.1:8080; echo "exit=$?"
```

```
000
exit=7
```

Exit code 7 is curl's "failed to connect". The packet did arrive at the host — but nothing told the host that port 8080 meant anything special, so it was delivered locally to the host's own port 8080, where nothing is listening, and the kernel answered with a TCP reset. Remember this failure; we're about to make the identical command succeed.

Now the rule:

```bash
sudo iptables -t nat -A PREROUTING -p tcp --dport 8080 -j DNAT --to-destination 10.20.1.2:8080
```

Two bits of syntax that trip people up. `--dport` is not a core iptables match; it's provided by the tcp match extension, so **`-p tcp` must come first** or you get `unknown option "--dport"`. And `--to-destination` is an option *of* the DNAT target, so it has to come after `-j DNAT` — put it before and iptables doesn't know what target it belongs to.

`PREROUTING` is the chain traversed as soon as a packet arrives, **before** the routing decision. That ordering is the whole point: the rewrite has to happen before the kernel decides where the packet is going, otherwise — exactly as we just watched — it gets routed to the host's own local address and delivered locally.

Run the identical command again:

```bash
sudo ip netns exec ns0 curl -s -m3 -o /dev/null -w '%{http_code}\n' http://10.40.1.1:8080; echo "exit=$?"
```

```
200
exit=0
```

Same client, same target address, same port. The only thing that changed is one rule in `PREROUTING`, and that's the whole of port forwarding. Confirm the rule is what did it:

```bash
sudo iptables -t nat -L PREROUTING -n -v
```

The DNAT rule's counter has moved. (One connection, one packet counted — the NAT table sees only the first packet of a flow, exactly as with MASQUERADE earlier.)

One honest caveat about this test: `ns0` could always reach `ns1` directly at `10.20.1.2:8080` over the bridge, so it isn't a stand-in for a machine that has *no* path to the namespace network. It's a convenient way to generate traffic that genuinely arrives at the host from outside the host's own stack, which is the property the DNAT rule needs. A second VM on `10.40.1.0/24` would be the fully realistic test, and it would behave the same way.

### The trap: testing this from the host itself

Now the test you'd probably have reached for first:

```bash
# on the VM itself — the host's own eth0 address
curl -s -m3 -o /dev/null -w '%{http_code}\n' http://10.40.1.1:8080; echo "exit=$?"
```

That fails, even though the same URL just worked from `ns0`. The reason is a classic and it wastes an enormous amount of people's time.

**Locally-generated traffic does not traverse `PREROUTING`.** That chain is for packets arriving from the network. When a process on the host itself opens a connection, the packet is created locally and goes through the `OUTPUT` chain instead, then `POSTROUTING`. Our rule is in `PREROUTING`, so it is never consulted, and `curl` ends up trying to connect to port 8080 on the host itself, where nothing is listening.

Check the counter and you'll see it plainly — it hasn't moved since the `ns0` test. The rule isn't broken; it was never reached.

The fix is to give locally-generated traffic the same treatment, with the equivalent rule in `OUTPUT`:

```bash
sudo iptables -t nat -A OUTPUT -p tcp -d 10.40.1.1 --dport 8080 -j DNAT --to-destination 10.20.1.2:8080
```

Note the `-d 10.40.1.1` — without it, this rule would catch *every* local connection to port 8080 anywhere, including the one we made earlier to `10.20.1.2:8080`, which would be a mess. Now:

```bash
curl -s -m3 -o /dev/null -w '%{http_code}\n' http://10.40.1.1:8080; echo "exit=$?"
```

`200`. And check both counters:

```bash
sudo iptables -t nat -L -n -v
```

Both rules now have non-zero counters: `PREROUTING` from the `ns0` test, `OUTPUT` from the `curl` we just ran on the host. That's the honest picture — `PREROUTING` is the rule that matters for real external traffic, `OUTPUT` is the one that makes the host itself behave the same way, and you generally want both if the host is also a client of its own forwarded ports. This is exactly why Docker installs rules in both chains for a published port.

Watch it happen if you like:

```bash
# second session
sudo tcpdump -i br-ns -n tcp port 8080
```

You'll see the connection arriving at `10.20.1.2:8080` over the bridge — the destination already rewritten by the time it reaches the namespace, which sees a perfectly ordinary connection and knows nothing about the translation.

When you're done, stop the `http.server` with Ctrl-C.

## Wrapping up

Let's recap the shape of what we built.

- A **network namespace** is an independent copy of the network stack — its own interfaces, routing table, ARP table, iptables rules, and port space. A fresh one contains only a down `lo` and can't even ping itself. Isolation is the default; connectivity is what you build.
- A **veth pair** is a virtual cable: two interfaces created together, wired back-to-back. Move one end into a namespace, bring **both** ends up, and traffic flows. Deleting either end deletes both.
- **Prefix lengths matter.** `ip addr add 10.20.1.1 dev veth-ns0` without `/24` gives you a `/32`, no subnet route, and a failure that looks like broken wiring.
- A **Linux bridge** is a virtual switch — it learns MACs per port and forwards frames. It turns N(N-1)/2 cables into N, which is the only reason this scales past two namespaces. Bridge-side veth ends must be brought up explicitly; `master br-ns` attaches them but does not enable them.
- Giving the bridge an address (`10.20.1.5/24 dev br-ns`) puts the **host on the namespace network**, and since it's also on `10.40.1.0/24` via `eth0`, it's now a machine with a leg in two networks — which is what a router is.
- **`Network is unreachable` means no route**; nothing left the machine. A ping that goes out and gets nothing back is a completely different failure, and `ping` can't tell them apart. `tcpdump` at an intermediate point can.
- **`net.ipv4.ip_forward`** must be 1 or the host won't pass traffic between its interfaces at all. Necessary but not sufficient. Note that it only governs traffic being *passed through* — packets addressed to one of the host's own addresses are delivered locally and need no forwarding, which is why picking a test target that's secretly the host itself hides this lesson entirely.
- The other half is **NAT**. `MASQUERADE` in `POSTROUTING` rewrites the source to the outgoing interface's address, and **conntrack** reverses it on the reply. It exists here because `10.20.1.0/24` is private to this VM — nothing outside has a route back to it, and being RFC 1918 space, no internet router would carry it anyway.
- A **default route** (`0.0.0.0/0`) catches everything not matched by a more specific route. Routing always prefers the most specific match — the longest prefix containing the destination, independent of the order routes were added.
- **DNAT** in `PREROUTING` forwards an external port into a namespace. `-p tcp` before `--dport`, `--to-destination` after `-j DNAT`. And **locally-generated traffic never traverses `PREROUTING`** — it goes through `OUTPUT`, which is why testing a port-forward from the host itself fails unless you add the matching `OUTPUT` rule.

None of this survives a reboot, and that's deliberate — it kept the moving parts visible. Namespaces created with `ip netns` live in `/var/run/netns` and vanish on reboot along with every interface, address, route, and iptables rule we created. In the real world:

- Addresses and routes on Ubuntu are declared in netplan YAML under `/etc/netplan/`; `net.ipv4.ip_forward` belongs in `/etc/sysctl.d/`.
- iptables rules persist via `iptables-save` / `iptables-restore`, usually through the `iptables-persistent` package, and on modern systems you'd more likely be writing `nftables` rules directly.
- `systemd-networkd` can create and manage bridges and veth pairs declaratively with `.netdev` and `.network` files.
- But most of the time you would not be doing any of this by hand at all, because a container runtime is doing it for you.

Which is the real payoff here. Run `ip link show` on any machine with Docker installed and you'll find `docker0` — a bridge, exactly like our `br-ns`, with an address on the container network. Start a container and a veth pair appears, one end inside the container's network namespace, the other attached to `docker0`. Outbound traffic gets a MASQUERADE rule in `POSTROUTING` for the container subnet. Publish a port with `-p 8080:8080` and you get a DNAT rule in `PREROUTING` pointing at the container's address, plus a companion rule in `OUTPUT` so it works from the host too. Every single piece we built by hand in this tutorial is there, generated automatically, and `iptables -t nat -L -n -v` on a Docker host will show you rules you can now read line by line.

To tear everything down, inside the VM:

```bash
sudo ip netns del ns0
sudo ip netns del ns1
sudo ip link del br-ns

sudo iptables -t nat -D POSTROUTING -s 10.20.1.0/24 -j MASQUERADE
sudo iptables -t nat -D PREROUTING -p tcp --dport 8080 -j DNAT --to-destination 10.20.1.2:8080
sudo iptables -t nat -D OUTPUT -p tcp -d 10.40.1.1 --dport 8080 -j DNAT --to-destination 10.20.1.2:8080

sudo sysctl -w net.ipv4.ip_forward=0
sudo rm -rf /etc/netns/ns1
```

Deleting a namespace takes its interfaces with it, and deleting the bridge takes the bridge-side veth ends with it, so there's nothing left over. To delete an iptables rule you give `-D` with the exact same specification you used for `-A`.

And then the VM and the network:

```bash
limactl stop nshost
limactl delete nshost
limactl network delete --force nsnet
```

`limactl network delete` requires `--force` if any instance has referenced the network.
