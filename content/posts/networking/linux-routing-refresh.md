---
title: "Linux Networking Refresh: Routing"
description: "A hands-on refresh on Linux routing. We create two isolated networks and three VMs with Lima, assign addresses by hand, and work through each failure — no route, no forwarding, no return route — until traffic flows between the two networks through a router."
date: 2026-09-09
lastmod: 2026-09-09
draft: true
sidebar: "right"
widgets:
  - "ddg-search"
  - "recent"
  - "social"
tags:
  - "key concepts"
  - "networking"
  - "linux"
  - "routing"
  - "lima"
  - "tcpdump"
---

In this tutorial we are going to do a refresh on routing in Linux networking. To simulate an environment we will use Linux machines via [Lima](https://lima-vm.io/) and the `limactl` CLI. We will create two virtual machines on two different virtual networks and establish communication between them via a router — a third virtual machine that we will also create.

This is the first part of a series. Here we get routing working between real (virtual) machines; a follow-up will build on the same concepts with network namespaces and container networking.

<!--more-->

To follow along, you should be able to use Lima from any OS you might be using. See the [installation guide](https://lima-vm.io/docs/installation/).

This tutorial was created with Lima 2.2.0.

```bash
limactl --version
limactl version 2.2.0
```

## Router Topology with Linux Machines

Note that our goal here is not to dive into Lima but to use it as supporting infrastructure for the networking concepts. For this reason, I will give most of the commands without much explanation, unless they add to our main goal here. To learn more about Lima, check its [documentation](https://lima-vm.io/docs/) or the `limactl` help.

Create the two isolated networks:

```bash
limactl network create net1 --gateway 10.20.1.1/24
limactl network create net2 --gateway 10.20.2.1/24
```

You can check them with:

```bash
limactl network list
```

I'm omitting the output, but you should see some default networks (`bridge`, `shared`, etc.) and the ones we just created: `net1` and `net2`, listed with `MODE: user-v2` — these are software-defined networks.

Launch the two VMs that we want to establish communication between:

```bash
limactl create --name=node1 --network=lima:net1 template:ubuntu -y
limactl create --name=node2 --network=lima:net2 template:ubuntu -y
```

Note that we are providing the `--network` parameter to pass the networks we created, one for each VM.

Now create the VM that will work as the router:

```bash
limactl create --name=router --network=lima:net1 --network=lima:net2 template:ubuntu -y
```

The main difference is that now we are providing two networks to the `router` machine. The router will have two network interfaces, one in each network. We will take a closer look at this in a bit.

In Lima, after you create the machines you have to start them. You can do so with the `limactl start` command:

```bash
limactl start node1
limactl start node2
limactl start router
```

To connect/ssh into a machine we will use `limactl shell <machine name>`. Sometimes, for convenience, we will execute commands directly from the host to inspect using `limactl shell <machine name> -- <command>`.

So, let's go ahead and check our current network setup by running `ip -4 addr show` on the three machines:

```bash
limactl shell node1 -- ip -4 addr show
limactl shell node2 -- ip -4 addr show
limactl shell router -- ip -4 addr show
```

You should see `node1` with a NIC called `eth0` on network `10.20.1.0/24`, `node2` also with a NIC `eth0` but on network `10.20.2.0/24`, and finally `router` with two NICs: `eth0` probably on `10.20.1.0/24` and `lima1` on `10.20.2.0/24`. All four NICs got IPs assigned to them, and that's coming from Lima's DHCP. Don't worry too much about the specific addresses that were handed out. We could work with them, but to practice and to have more control over the setup, we'll flush the existing IPs and reassign them. If we were working on a network that doesn't have a DHCP lease server, we would have to assign them manually anyway.

## Configuring IPs

In this section we will use `ip addr` to flush the assigned IPs and assign new ones.

Important to note that everything we do here is not persistent. It's a good way to understand the setup step by step conceptually, instead of me just throwing a bunch of YAML at you.

```bash
# ssh into node1
limactl shell node1

# check
ip -4 addr show

# flush eth0
sudo ip addr flush dev eth0

# check again
ip -4 addr show

# add the new address
sudo ip addr add 10.20.1.10/24 dev eth0

# check again
ip -4 addr show
```

Ok, now we have `node1` on IP `10.20.1.10`.

Do the same for `node2`:

```bash
# ssh into node2
limactl shell node2

# check
ip -4 addr show

# flush eth0
sudo ip addr flush dev eth0

# check again
ip -4 addr show

# add the new address
sudo ip addr add 10.20.2.10/24 dev eth0

# check again
ip -4 addr show
```

Now the router. I will skip all the intermediate checks this time, but feel free to run them.

```bash
limactl shell router
sudo ip addr flush dev eth0
sudo ip addr add 10.20.1.1/24 dev eth0
sudo ip addr flush dev lima1
sudo ip addr add 10.20.2.1/24 dev lima1

# check everything is as expected
ip -4 addr show
```

Note that we're giving it `.1` on each network. That's just a widespread convention — routers are commonly placed at the first address of a subnet, though `.254` is also common, and any free address would work just as well. Nothing about being a router depends on the address itself. What makes this machine a router is that it has a leg in both networks and forwards traffic between them, and the only thing that matters about its addresses is that the routes we add later point at them.

## Routing

Now we have the following:

- `node1` IP: `10.20.1.10` (`net1`)
- `node2` IP: `10.20.2.10` (`net2`)
- `router` IPs: `10.20.1.1` on `net1` and `10.20.2.1` on `net2`

Let's do some connectivity checks:

```bash
limactl shell node1

# check connectivity to node2 (it will fail)
ping -c2 10.20.2.10

# now check connectivity to the router on net1's IP
ping -c2 10.20.1.1
```

When pinging `node2` you will get `connect: Network is unreachable`. Pinging the router's IP on `net1` will work because they are on the same network. Note that pinging the router on its `net2` IP (`10.20.2.1`) would fail right now too, and for exactly the same reason as `node2` — it's an address on `10.20.2.0/24`, and `node1` currently has no route to that subnet at all. It doesn't matter that the router is directly attached to `node1` on the other side; as far as `node1`'s routing table is concerned, that whole network doesn't exist yet.

Now let's add the routing configuration.

On `node1`:

```bash
limactl shell node1
sudo ip route add 10.20.2.0/24 via 10.20.1.1 dev eth0

# you can check the route added with
ip route show
```

Now try pinging `node2`:

```bash
ping -c2 10.20.2.10
```

Note that you don't receive the `Network is unreachable` error anymore, but you don't get any packets back either.

Try a similar test, but reaching the router's IP on `net2`:

```bash
ping -c2 10.20.2.1
```

This works, right? What's happening? Pinging `10.20.2.1` succeeds because that traffic is addressed **to** the router itself — it arrives on `eth0` and the router answers it locally, no forwarding involved. Reaching `node2` is different: the router has to accept a packet on `eth0` that's destined for a different machine and pass it out `lima1`. That's forwarding, and by default Linux won't do it — a host only handles traffic meant for itself unless you explicitly turn it into a router by setting `net.ipv4.ip_forward` to 1. Note that this is necessary but not sufficient. There's one more thing missing, and I want you to find it rather than take my word for it.

So, on the `router`:

```bash
limactl shell router

# verify the current ip_forward value
cat /proc/sys/net/ipv4/ip_forward

sudo sysctl -w net.ipv4.ip_forward=1
```

Now, back to `node1`:

```bash
limactl shell node1

ping -c2 10.20.2.10
```

Still doesn't work, right? And at this point `ping` has told us everything it can. It only knows "no reply came back", and that's the same output whether the request never reached `node2` at all or it arrived fine and the reply got lost on the way home. Those are very different problems with the same symptom, and no amount of pinging from `node1` will tell them apart. To separate them we have to stop asking the endpoints and go watch the traffic in the middle.

So let's do that: open two additional sessions in your terminal and keep them visible (ideally, but not necessarily). Run the following commands, one in each of these new sessions. You will watch the traffic on `router` and `node2`.

```bash
limactl shell router -- sudo tcpdump -i any -n icmp
limactl shell node2 -- sudo tcpdump -i eth0 -n icmp
```

Now ping `node2` from `node1` again. You should see the echo request flowing from `node1` to `node2` through the router — and on `node2`'s own capture, the request arriving. But no reply comes back out.

So what does that tell us? The request is getting all the way there. `node2` is receiving it. The problem is on the way back, and it's `node2`'s problem. Before reading on, have a look at its routing table:

```bash
limactl shell node2 -- ip route show
```

`node2` only knows about its own subnet. It has no route to `10.20.1.0/24`, so when it tries to answer `node1` it has nowhere to send the reply — the same situation `node1` was in at the very beginning, just in the opposite direction. Let's add that route now.

On `node2`:

```bash
limactl shell node2
sudo ip route add 10.20.1.0/24 via 10.20.2.1 dev eth0

# you can check the route
ip route show
```

Now if you go back to `node1` and test again, you'll see the connectivity is established, with the requests and replies logged on `router` and `node2`. Of course, you could also have checked the ping directly from `node2` and verified it was already working, but since you were watching the two `tcpdump` outputs I thought it would be cool to go back and test from the baseline.

## Wrapping up

Let's recap what we did. We created two isolated networks and three machines: `node1` on `net1`, `node2` on `net2`, and a `router` with one interface on each. We assigned addresses by hand, then worked through why connectivity failed at each stage:

- **No route:** `node1` had no idea `10.20.2.0/24` existed — `Network is unreachable`. Adding `10.20.2.0/24 via 10.20.1.1` told it where to send those packets.
- **No forwarding:** the router received the packets but wouldn't pass them between its interfaces until `net.ipv4.ip_forward` was set to 1.
- **No return route:** `node2` received the echo requests but had no route back to `10.20.1.0/24`, so the replies never made it home. `ping` couldn't tell us this on its own — it took `tcpdump` at an intermediate point to show that the requests were arriving fine and only the reply direction was broken.

As I mentioned before, none of this configuration survives a reboot — it was all done with `ip` commands against the live kernel state, which is exactly what makes it good for learning. In the real world you wouldn't configure this by hand.

On Ubuntu (used here), addresses and routes are declared in netplan YAML under `/etc/netplan/`, which persists them across reboots; `net.ipv4.ip_forward` would go in `/etc/sysctl.d/` instead. Other distros use different tooling (NetworkManager on RHEL-family systems, for example), but the underlying kernel state is the same — that's what we've been manipulating directly with `ip`.

In the next part we'll apply the same ideas to network namespaces and container networking.

To tear everything down

```bash
limactl stop node1
limactl stop node2
limactl stop router

limactl delete node1
limactl delete node2
limactl delete router

limactl network delete --force net1
limactl network delete --force net2
```