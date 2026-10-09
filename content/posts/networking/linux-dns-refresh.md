---
title: "Linux Networking Refresh: DNS"
description: "A hands-on refresh on DNS. Starting from the plainest fact - a machine is reachable by its address - we work up through /etc/hosts, nsswitch.conf, and resolv.conf, stand up real servers with dnsmasq and CoreDNS, walk the recursive hierarchy with dig +trace, tour the record types you actually debug, and then break things on purpose to work them out with the tools."
date: 2026-09-27
lastmod: 2026-09-27
draft: false
categories:
  - "Key Concepts"
tags:
  - "networking"
  - "linux"
  - "dns"
  - "dnsmasq"
  - "coredns"
  - "systemd-resolved"
  - "dig"
  - "lima"
  - "tcpdump"
---

Continuing the series of [Linux Networking](https://deployment.properties/tags/networking/), we are going to do a refresh on DNS. We will start from the plainest possible fact — a machine is reachable by its address — and work our way up through `/etc/hosts`, `nsswitch.conf`, `/etc/resolv.conf`, a real DNS server we run ourselves, the recursive hierarchy that answers queries for the rest of the internet, the record types you actually meet in the wild, and the tools you use when any of it goes wrong.

<!--more-->

To simulate an environment we will use Linux machines via [Lima](https://lima-vm.io/) and the `limactl` CLI. DNS doesn't need a complicated topology, so we'll keep it to one network and two machines: a `dnsserver` and a `client`.

To follow along, you should be able to use Lima from any OS you might be using. See the [installation guide](https://lima-vm.io/docs/installation/).

This tutorial was created with Lima 2.2.0.

```bash
limactl --version
limactl version 2.2.0
```

## The setup

Note that our goal here is not to dive into Lima but to use it as supporting infrastructure for the networking concepts. For this reason, I will give most of the commands without much explanation, unless they add to our main goal. To learn more about Lima, check its [documentation](https://lima-vm.io/docs/) or the `limactl` help.

Create one isolated network:

```bash
limactl network create dnsnet --gateway 10.30.1.1/24
```

You can check it with:

```bash
limactl network list
```

I'm omitting the output, but you should see some default networks (`bridge`, `shared`, etc.) and the one we just created: `dnsnet`, listed with `MODE: user-v2` - a software-defined network.

Now create the two VMs:

```bash
limactl create --name=dnsserver --network=lima:dnsnet template:ubuntu -y
limactl create --name=client --network=lima:dnsnet template:ubuntu -y
```

Start them:

```bash
limactl start dnsserver
limactl start client
```

To connect/ssh into a machine we will use `limactl shell <machine name>`. Sometimes, for convenience, we will execute commands directly from the host using `limactl shell <machine name> -- <command>`.

Let's look at what addresses we got:

```bash
limactl shell dnsserver -- ip -4 addr show
limactl shell client -- ip -4 addr show
```

We'll flush the IPs assigned by DHCP and assign addresses by hand, so that the rest of the tutorial is deterministic:

```bash
# on dnsserver
limactl shell dnsserver
sudo ip addr flush dev eth0
sudo ip addr add 10.30.1.53/24 dev eth0
ip -4 addr show eth0
```

```bash
# on client
limactl shell client
sudo ip addr flush dev eth0
sudo ip addr add 10.30.1.20/24 dev eth0
ip -4 addr show eth0
```

`10.30.1.53` for the DNS server is a mnemonic - port 53 is DNS - but there is nothing special about it. Nothing about being a DNS server depends on the address.

Important to note that everything we do here, both the addressing and most of the DNS config later, is not persistent. It's a good way to understand the setup step by step conceptually.

One more thing: flushing `eth0` also dropped the default route on each machine, so the VMs can't reach the internet right now. We'll need that later for `dig +trace`, so let's put a default route back, pointing at the network's gateway:

```bash
# on both machines
sudo ip route add default via 10.30.1.1 dev eth0
```

If `dnsserver` ended up owning `10.30.1.1` from DHCP earlier, don't worry - we flushed it, and `10.30.1.1` is the network's gateway again.

Verify the two machines can see each other:

```bash
limactl shell client -- ping -c2 10.30.1.53
```

## Reachability by address

That ping is the whole point of the first section. `client` reached `dnsserver`, and not one byte of DNS was involved. The kernel had a route, ARP found the MAC, the packets went out. **An address is what makes a machine reachable.** Names are not.

Let's make it a little more concrete by putting something on `dnsserver` worth talking to:

```bash
limactl shell dnsserver
sudo apt-get update
sudo apt-get install -y nginx dnsutils tcpdump
```

And from `client`:

```bash
limactl shell client
sudo apt-get update
sudo apt-get install -y dnsutils curl tcpdump ldnsutils

curl -sI http://10.30.1.53 | head -1
```

You should see `HTTP/1.1 200 OK`. Again, no names anywhere.

So why do we bother with DNS at all? Two reasons, and only two really matter:

1. `10.30.1.53` is hard to remember and easy to mistype, and a human-facing system full of literal addresses is impractical to operate.
2. Addresses **change**. The machine gets rebuilt, moves to another subnet, scales to five replicas behind a load balancer. Every place that hard-coded the address is now wrong. A name is a level of indirection, and indirection is what lets the thing behind it move.

Everything that follows is a progressively better answer to "how do we map names to addresses".

## /etc/hosts - the oldest answer

Before DNS existed, the entire internet's name-to-address mapping lived in a single file called `HOSTS.TXT`, maintained at Stanford, and every machine downloaded a copy periodically. That file survives as `/etc/hosts`.

On `client`:

```bash
limactl shell client

# look at what's already there
cat /etc/hosts

# add our own entry
echo "10.30.1.53 web.lab.local web" | sudo tee -a /etc/hosts

# prove it took effect
ping -c2 web.lab.local
curl -sI http://web | head -1
```

The format is one line per address: the address first, then one or more names for it. The first name is conventionally the canonical one and the rest are aliases - that's why `web` works as well as `web.lab.local`.

That worked instantly. No server, no network round trip, no daemon to restart. And that's exactly the problem:

- It is **purely local**. Go to `dnsserver` and run `ping web.lab.local` - it fails with `Name or service not known`. Nothing you did on `client` is visible anywhere else.
- There is **no TTL**. The entry is true until someone edits the file. There's no notion of "this answer is good for 5 minutes", which means no controlled way to change it.
- There is **no delegation**. You cannot say "I own `lab.local`, but ask someone else about `eu.lab.local`". It's a flat list.
- It **does not scale**. Every machine needs its own copy, and every change has to be pushed to every machine. That's fine for 3 hosts and unworkable for 300.

`/etc/hosts` is still genuinely useful - for pinning a hostname during a migration, for local development overrides, for breaking a dependency on DNS while you debug DNS. But the list above is the entire motivation for the rest of this tutorial.

## /etc/nsswitch.conf - who decides that hosts wins?

When you ran `ping web.lab.local`, how did `ping` know to look in `/etc/hosts` rather than ask a DNS server? That behaviour is configurable.

On `client`:

```bash
grep '^hosts' /etc/nsswitch.conf
```

You should see something close to:

```
hosts:          files dns
```

That line is the answer. `/etc/nsswitch.conf` - the Name Service Switch - tells glibc, for each kind of lookup (users, groups, hosts, ...), which backends to consult and **in what order**. For `hosts`, `files` means `/etc/hosts` and `dns` means "go do a real DNS query using `/etc/resolv.conf`". They're tried left to right, and the first one that answers wins. On some Ubuntu images you'll see extra entries like `mdns4_minimal [NOTFOUND=return]` or `resolve [!UNAVAIL=return]` - those are additional backends (multicast DNS, systemd-resolved's own bus API) with rules about when to stop.

So `/etc/hosts` doesn't beat DNS because it's special. It beats DNS because `files` comes before `dns` on that line. Swap the order and DNS would win.

### Why `ping` and `dig` disagree - the classic gotcha

This is a common source of confusion in DNS debugging, so let's reproduce it. Still on `client`:

```bash
ping -c1 web.lab.local
dig +short web.lab.local
```

`ping` resolves it. `dig` returns nothing at all.

The reason is that they are doing fundamentally different things. `ping`, `curl`, `ssh`, your application - all of them resolve names by calling `getaddrinfo()` in glibc, which follows `/etc/nsswitch.conf`, which consults `/etc/hosts` first. **`dig` does none of that.** `dig` is a DNS protocol tool: it builds a DNS query packet, sends it to a nameserver over UDP or TCP, and prints the response. It never reads `/etc/hosts` and it never consults nsswitch. The only thing it takes from `/etc/resolv.conf` is which server to talk to.

**`dig` tells you what DNS says. It does not tell you what your application will do.** When someone reports "the app can't resolve it but dig works fine" (or the reverse), the difference between these two paths is almost always the answer. `getent hosts web.lab.local` is the tool that follows the same path your application does, and it's the right thing to compare against:

```bash
getent hosts web.lab.local
```

That one *will* return `10.30.1.53`, because `getent` goes through nsswitch just like `ping` does.

## /etc/resolv.conf - where DNS queries go

When nsswitch does get to `dns`, the resolver needs to know which server to ask. That comes from `/etc/resolv.conf`.

```bash
ls -l /etc/resolv.conf
cat /etc/resolv.conf
```

Two things to notice. First, on a modern Ubuntu it is very likely a **symlink** - something like `/etc/resolv.conf -> ../run/systemd/resolve/stub-resolv.conf` - and its content is probably:

```
nameserver 127.0.0.53
options edns0 trust-ad
search .
```

Second, the nameserver is `127.0.0.53`, which is not a real DNS server out on the network. It's the **stub resolver** of `systemd-resolved`, a local daemon listening on loopback. Your application's query goes to `127.0.0.53`, `systemd-resolved` decides what to do with it (cache hit? which upstream? which link's DNS servers?), and forwards it on. We'll deal with this in a moment, because it has real consequences.

The directives you'll actually use:

- `nameserver <ip>` - a server to query. You can list up to three. They are tried **in order**, and the second one is only used when the first fails to answer (times out), not to load-balance. A dead first nameserver means every lookup eats a timeout before succeeding.
- `search <domain> [<domain> ...]` - the search list. If you look up a name with fewer dots than `ndots` (default 1), the resolver appends each search domain in turn and tries those first. With `search lab.local`, looking up `web` tries `web.lab.local.` before trying `web.` on its own.
- `domain <domain>` - an older, single-entry form of `search`. If both appear, the last one in the file wins.
- `options ndots:N` - how many dots a name needs before it's tried as-is first, rather than being run through the search list. Kubernetes sets `ndots:5`, which is why an in-cluster lookup of `api.github.com` can generate four failed queries before the right one.
- `options timeout:N` / `options attempts:N` - per-server timeout in seconds and how many rounds to try. Worth knowing when a hang is 5 seconds versus 30.
- `options edns0` - enable EDNS(0), the extension mechanism that allows UDP responses larger than 512 bytes (among other things). Without it, larger answers force a retry over TCP.

The practical consequence of the stub resolver: if you run `tcpdump port 53` on `client`'s `eth0` while an application resolves a name, you will see **nothing**, because the query went to loopback. The traffic you want is either on the `lo` interface, or it's the query `systemd-resolved` makes upstream on your behalf. Use `-i any` when you're not sure:

```bash
# on client, in a second session
sudo tcpdump -i any -n port 53
```

Then run a lookup in your first session and see where the packets actually appear.

## Standing up a real DNS server with dnsmasq

Let's now do the thing `/etc/hosts` couldn't: serve names to *other* machines, from one place, with TTLs.

We'll use `dnsmasq` - a small, extremely widespread server that does DNS forwarding, caching, and a bit of authoritative serving, and is what's inside a great many home routers and lightweight setups.

On `dnsserver`:

```bash
limactl shell dnsserver
sudo apt-get install -y dnsmasq
```

Then check whether it actually started:

```bash
systemctl status dnsmasq --no-pager
```

There's a decent chance it's failed, and the reason is worth understanding:

```bash
sudo ss -lunp | grep :53
```

### The port 53 conflict

`systemd-resolved` is already listening on port 53 - on `127.0.0.53` specifically. `dnsmasq` by default tries to bind port 53 on **all** addresses, which includes `127.0.0.53`, and the bind fails. Two daemons, one port.

We're building a DNS server here, and having a second resolver on the box confusing matters is not what we want, so let's be decisive and disable `systemd-resolved` on `dnsserver`:

```bash
# on dnsserver
sudo systemctl disable --now systemd-resolved

# resolv.conf is now a dangling symlink to a file that will never be written again;
# replace it with a real file so the machine can still resolve public names
sudo rm -f /etc/resolv.conf
echo "nameserver 8.8.8.8" | sudo tee /etc/resolv.conf

sudo systemctl restart dnsmasq
systemctl status dnsmasq --no-pager
```

It should be `active (running)` now.

You don't *have* to disable `systemd-resolved`. You can instead tell it to release port 53 with `DNSStubListener=no` in `/etc/systemd/resolved.conf`, or tell `dnsmasq` to bind only specific interfaces with `bind-interfaces` plus `listen-address=`. Both are legitimate. Disabling it outright is the clearest thing to do on a box whose entire job is being a nameserver.

### Configuring the zone

`dnsmasq` reads `/etc/dnsmasq.conf` and, on Ubuntu, everything in `/etc/dnsmasq.d/`. We'll put our config in its own file:

```bash
# on dnsserver
sudo tee /etc/dnsmasq.d/lab.conf >/dev/null <<'EOF'
# don't read /etc/resolv.conf for upstream servers; be explicit
no-resolv
server=8.8.8.8
server=1.1.1.1

# we are authoritative-ish for lab.local: never forward these upstream
local=/lab.local/

# our records
address=/web.lab.local/10.30.1.53
address=/db.lab.local/10.30.1.20
address=/ns.lab.local/10.30.1.53

# how long clients may cache our answers, in seconds
local-ttl=60

# cache size (entries) and logging so we can watch what happens
cache-size=1000
log-queries
EOF

sudo systemctl restart dnsmasq
```

A few notes on what those directives mean:

- `no-resolv` stops dnsmasq from picking up upstream servers out of `/etc/resolv.conf`. Without it, and with a misconfigured `resolv.conf`, you can trivially make dnsmasq forward queries to itself. That loop is a common misconfiguration.
- `local=/lab.local/` says: names under `lab.local` are mine. If I don't have an answer, reply `NXDOMAIN` - do **not** ask upstream. This is what makes the zone private rather than leaky.
- `address=/name/ip` is dnsmasq's shorthand for an A record. `address=/lab.local/10.30.1.53` (no host part) would match the domain *and every name under it*, which is useful for wildcards and easy to trigger unintentionally.
- `local-ttl` sets the TTL dnsmasq puts on its own answers. The default is 0, meaning "do not cache this", which would hide the caching behaviour we want to observe.

### Pointing the client at it

On `client`, we hit the same `systemd-resolved` question from the other side. If we just edit `/etc/resolv.conf`, `systemd-resolved` (or its networking integration) will happily overwrite it on the next network event, and even when it doesn't, our queries still go through the stub at `127.0.0.53`.

We have two options. The systemd-native one is to tell `resolved` about our server for the link:

```bash
# on client - the systemd-resolved way
sudo resolvectl dns eth0 10.30.1.53
sudo resolvectl domain eth0 lab.local
resolvectl status eth0
```

This keeps the stub at `127.0.0.53`, with `resolved` forwarding `lab.local` (and, given the config above, everything else on that link) to `10.30.1.53`. It works, and it's what you'd do on a real systemd host. But it means every packet capture on `client` shows loopback traffic, and it inserts a second cache between you and the thing you're trying to observe.

Since this is a tutorial about seeing DNS work, we'll take the blunter option and get `resolved` out of the path entirely:

```bash
# on client - the blunt way
sudo systemctl disable --now systemd-resolved
sudo rm -f /etc/resolv.conf
sudo tee /etc/resolv.conf >/dev/null <<'EOF'
nameserver 10.30.1.53
search lab.local
options edns0
EOF

cat /etc/resolv.conf
```

Now `client` sends its queries straight to `dnsserver`, and `search lab.local` means short names get the domain appended automatically.

Before we test, remove the `/etc/hosts` entry we added earlier - otherwise `files` will keep winning and we'll never reach DNS at all:

```bash
sudo sed -i '/web.lab.local/d' /etc/hosts
grep web /etc/hosts || echo "gone"
```

### Watching queries arrive

Open a second session and start a capture on the server:

```bash
limactl shell dnsserver -- sudo tcpdump -i any -n port 53
```

And a third, to watch dnsmasq's own log:

```bash
limactl shell dnsserver -- sudo journalctl -u dnsmasq -f
```

Now, from `client`:

```bash
dig web.lab.local
```

You should get a `status: NOERROR` and an ANSWER section with `web.lab.local. 60 IN A 10.30.1.53`. On the tcpdump you'll see the query arrive from `10.30.1.20` and the response go back; in the journal you'll see dnsmasq log the query and the reply it constructed.

Now confirm the plumbing works for real applications too, not just for `dig`:

```bash
ping -c2 web
curl -sI http://web.lab.local | head -1
```

`ping web` works because of the `search lab.local` line - the resolver appended the domain. This is also the first thing to suspect when a short name resolves on one machine and not another.

### Caching and TTL

Ask for something dnsmasq has to fetch from upstream, twice in a row, and watch both the log and the reported TTL:

```bash
# on client
dig +noall +answer example.com
sleep 5
dig +noall +answer example.com
```

The first query shows a TTL - say `86400`. The second shows a **smaller** TTL, roughly the first minus the seconds that passed. That decrementing TTL is the signature of a cached answer: dnsmasq is counting down the remaining lifetime of the record it stored, not re-fetching it. In the journal you'll see the first query logged as forwarded upstream and the second logged as answered from the cache. On the tcpdump, the first produces outbound traffic from `dnsserver` to `8.8.8.8`; the second produces none at all.

That's the whole idea of a **TTL**: the authoritative owner of a record states how long anyone may hold onto the answer. Short TTLs mean fast changes and more query load; long TTLs mean less load and slower propagation. When you plan a migration, you lower the TTL *first*, wait for the old one to expire everywhere, then make the change.

You can also inspect dnsmasq's cache statistics directly - it dumps them to the log on `SIGUSR1`:

```bash
# on dnsserver
sudo pkill -USR1 dnsmasq
sudo journalctl -u dnsmasq -n 20 --no-pager
```

You'll see counts for cache size, insertions, and - the number that matters - hits versus misses.

## Recursion and the hierarchy

Notice what just happened with `example.com`: `client` asked `dnsserver`, and `dnsserver` did not have the answer. It went and got it. That behaviour is called **recursion**, and it's worth being precise about, because this is the concept people most often have muddled.

There are two distinct roles:

- An **authoritative server** holds the actual zone data for some part of the namespace. It answers questions about names it owns, out of its own configured records, and it answers "I don't know, but ask over there" (a *referral*) for anything below a zone it has delegated. It never goes and asks anyone else on your behalf. Our `dnsmasq` is authoritative for `lab.local`.
- A **recursive resolver** owns no zone data at all. Its job is to take your one question and chase it down through the hierarchy, asking as many authoritative servers as it takes, then hand you the final answer and cache it. `8.8.8.8`, `1.1.1.1`, and your ISP's resolver are recursive resolvers. Our `dnsmasq` is *also* acting as one - it forwards, which is a lightweight variant, but the role is the same from the client's point of view.

The namespace itself is a tree, read right to left. `www.example.com.` - note the trailing dot, which is the root - decomposes as: root, then the `com` TLD, then `example.com`, then the `www` label within it. Each level **delegates** the level below it by publishing NS records pointing at the next servers down.

Let's watch a real resolution happen. `dig +trace` starts at the root and follows the referrals itself, printing each step, instead of asking a recursive resolver to do it invisibly:

```bash
# on client - this one needs to reach the public internet
dig +trace www.example.com
```

Read the output as a sequence of hops:

1. First a list of the 13 **root servers** (`a.root-servers.net.` through `m.root-servers.net.`), which dig has built in as a starting hint.
2. It asks a root server about `www.example.com`. The root doesn't know - it doesn't know anything about individual websites. It returns a **referral**: the NS records for `com.`, pointing at `a.gtld-servers.net.` and friends. Note there's no ANSWER section here, just AUTHORITY.
3. It asks a `com.` server. That server doesn't know either, but it knows who was delegated `example.com.` - another referral, this time to the nameservers for `example.com`.
4. It asks one of *those*, and finally gets an actual ANSWER with the A record.

Notice the marker dig prints on the final response: `;; Received ... from ...` for each hop, and the final answer will show the `aa` flag - **authoritative answer** - because it came from a server that genuinely owns that data.

Now contrast with a normal query:

```bash
dig www.example.com @8.8.8.8
```

One question, one answer. Look at the flags line: you'll see `ra` (recursion available) and `rd` (recursion desired), and you will *not* see `aa`. Google's resolver did all four steps above for you - or, more likely, had it cached - and the answer is not authoritative because Google doesn't own `example.com`.

You can ask a server to *not* recurse and see the difference directly:

```bash
dig +norecurse www.example.com @8.8.8.8
```

If it's in Google's cache, you get an answer without the `rd` flag set. If it isn't, you get an empty answer - the resolver was told "answer from what you have, don't go asking around" and it had nothing.

Try that same flag against a root server to see a pure referral, with no recursion involved at all:

```bash
dig +norecurse @a.root-servers.net www.example.com
```

No ANSWER section, an AUTHORITY section full of `com.` NS records, and an ADDITIONAL section with their addresses (those are **glue records** - the addresses of nameservers that you'd otherwise need DNS to look up, creating a chicken-and-egg problem).

## Record types

A DNS zone isn't just names to addresses. Each record has a type, and each type answers a different question. Here's the working set, with something you can run for each.

### A - name to IPv4 address

The record most people mean when they say "DNS".

```bash
dig +noall +answer example.com A
```

```
example.com.		251	IN	A	23.192.228.84
```

The fields are: name, TTL, class (`IN` for Internet, in practice the only one you'll see), type, and data.

### AAAA - name to IPv6 address

Same idea, 128-bit address. "Quad-A" because it's four times the size of an A record.

```bash
dig +noall +answer example.com AAAA
```

Note that A and AAAA are separate queries. A dual-stack client issues both, and `getaddrinfo()` merges the results - which is why an application can behave differently from `dig example.com` if only one of the two families is broken.

### CNAME - canonical name, i.e. an alias

A CNAME says "this name is really that other name; go resolve that instead".

```bash
dig +noall +answer www.github.com
```

You'll typically see a CNAME line followed by the A records of the target, because the resolver followed the chain for you and returned the whole thing.

Two rules about CNAMEs:

1. **A CNAME cannot coexist with other records at the same name.** If `www.example.com` is a CNAME, it cannot also have an A record, or an MX record, or a TXT record. The alias is total - the name *is* the other name, so it can't have data of its own. (The one universal exception is DNSSEC's own signing records.)
2. **A CNAME cannot exist at a zone apex.** The apex - `example.com` itself, with no label in front - must carry SOA and NS records by definition. Since rule 1 forbids anything alongside a CNAME, the apex can never be one. This is why you cannot naively CNAME `example.com` to a load balancer's hostname, and why providers invented non-standard workarounds (ALIAS, ANAME, CNAME flattening) that resolve the target and serve A records at the apex instead.

### MX - mail exchanger

Where to deliver mail for a domain. Each record has a **preference** number; lower is preferred, equal values are load-balanced.

```bash
dig +noall +answer gmail.com MX
```

```
gmail.com.	3600	IN	MX	5 gmail-smtp-in.l.google.com.
gmail.com.	3600	IN	MX	10 alt1.gmail-smtp-in.l.google.com.
```

The target of an MX record must be a hostname with an A/AAAA record - pointing an MX at an IP address, or at a CNAME, is invalid, though a lot of software tolerates the latter.

### NS - nameserver, i.e. delegation

Which servers are authoritative for a zone. These are the records that make the hierarchy work.

```bash
dig +noall +answer example.com NS
dig +noall +answer com NS | head -3
```

NS records exist in two places for the same zone: in the parent (the delegation) and in the child itself (the authoritative copy). When those two disagree, you get a DNS bug that only manifests for some users.

### SOA - start of authority

Exactly one per zone, at the apex. It carries the zone's administrative parameters.

```bash
dig +noall +answer example.com SOA
```

```
example.com.  3600 IN SOA ns.icann.org. noc.dns.icann.org. 2024081493 7200 3600 1209600 3600
```

The fields, in order, and why each matters:

- **MNAME** (`ns.icann.org.`) - the primary nameserver for the zone. Where dynamic updates go, and the source secondaries expect to pull from.
- **RNAME** (`noc.dns.icann.org.`) - the responsible party's email, with the `@` written as a dot. That's `noc@dns.icann.org`.
- **Serial** (`2024081493`) - the zone version. Secondaries compare their serial against the primary's; if the primary's is higher, they transfer the zone. Forget to bump it after editing and your changes never propagate. The `YYYYMMDDnn` convention above is common but not required.
- **Refresh** (`7200`) - how often a secondary checks the primary's serial.
- **Retry** (`3600`) - how soon to check again after a failed check.
- **Expire** (`1209600`) - how long a secondary keeps serving the zone when it can't reach the primary at all. After two weeks here, it stops answering rather than serving data of unknown staleness.
- **Minimum** (`3600`) - originally a default TTL; since RFC 2308 it is the **negative caching TTL**. It's how long a resolver may cache an NXDOMAIN for this zone. If you set it to 86400 and someone looks up a name before you create it, they'll keep getting NXDOMAIN for a day after you've fixed it.

### TXT - arbitrary text

Free-form strings attached to a name. Originally for human notes; in practice it's the universal extension point - SPF, DKIM, DMARC, and every "add this TXT record to prove you own the domain" flow.

```bash
dig +noall +answer google.com TXT
```

Each string is limited to 255 characters, though a record can hold several concatenated.

### PTR - reverse lookup, address to name

Forward DNS maps names to addresses. PTR does the reverse, and it does it with a trick: addresses are turned into names inside a special zone.

For IPv4, the octets are **reversed** and appended to `in-addr.arpa`. So `8.8.4.4` becomes `4.4.8.8.in-addr.arpa`:

```bash
dig +noall +answer 4.4.8.8.in-addr.arpa PTR
```

The reversal exists because DNS names get more specific right to left, while IP addresses get more specific left to right. Reversing them lets `8.8.8.in-addr.arpa` be delegated to whoever owns `8.8.8.0/24`, exactly like a normal subdomain.

`dig` will do the reversal for you with `-x`:

```bash
dig -x 8.8.4.4
```

IPv6 works the same way but with each **nibble** (hex digit) reversed and dot-separated under `ip6.arpa`, which produces considerably longer names.

Crucially, **forward and reverse are separate and need not agree.** Reverse DNS is delegated by whoever owns the address block, not whoever owns the name - which is why you can point any name you like at an IP but can't set its PTR unless your provider lets you. Mail servers check forward-confirmed reverse DNS; most other software does not.

### SRV - service location

Instead of "what's the address of this host", SRV answers "where does this *service* live", including port, priority, and weight. The name has a structured form: `_service._protocol.name`.

```bash
dig +noall +answer _sip._tcp.example.com SRV
dig +noall +answer _xmpp-server._tcp.jabber.org SRV
```

```
_xmpp-server._tcp.jabber.org. 900 IN SRV 30 30 5269 hermes2.jabber.org.
```

The four data fields are **priority** (lower wins, like MX), **weight** (proportional load-balancing among equal priorities), **port**, and **target**. It's how Active Directory clients find domain controllers and how Kubernetes exposes named service ports internally.

There are more - CAA (which certificate authorities may issue for a domain), DNSKEY/RRSIG/DS (DNSSEC), NAPTR, HTTPS/SVCB (the modern successor to SRV for web) - but the set above covers most of what you'll encounter.

## The same zone in CoreDNS

Let's do the whole thing again with a different implementation, so the concepts stay separate from the tool. **CoreDNS** is the DNS server that runs inside essentially every Kubernetes cluster. Architecturally it's quite different from dnsmasq: it's a chain of **plugins**, and a query is passed down the chain until a plugin handles it.

Stop dnsmasq first - one port 53 per machine, still:

```bash
# on dnsserver
sudo systemctl stop dnsmasq
```

Install CoreDNS. Ubuntu packages it, but the packaged version is often old; grab the release binary instead:

```bash
# on dnsserver
COREDNS_VERSION=1.14.6
ARCH=$(dpkg --print-architecture)   # arm64 on Apple Silicon, amd64 on Intel
curl -sLO "https://github.com/coredns/coredns/releases/download/v${COREDNS_VERSION}/coredns_${COREDNS_VERSION}_linux_${ARCH}.tgz"
tar xzf "coredns_${COREDNS_VERSION}_linux_${ARCH}.tgz"
sudo install -m 0755 coredns /usr/local/bin/coredns
coredns --version
```

CoreDNS is configured by a **Corefile**. Create a directory for it and the zone file:

```bash
sudo mkdir -p /etc/coredns
sudo tee /etc/coredns/Corefile >/dev/null <<'EOF'
lab.local:53 {
    file /etc/coredns/db.lab.local
    log
    errors
}

.:53 {
    forward . 8.8.8.8 1.1.1.1
    cache 30
    log
    errors
}
EOF
```

Read that as two **server blocks**. The first says: for anything under `lab.local`, run this plugin chain - `file` (serve the zone from a zone file), `log`, `errors`. The second is the catch-all: for everything else, `forward` upstream and `cache` answers for up to 30 seconds. Plugin order inside a block is *not* the order you wrote it; CoreDNS has a fixed, compiled-in ordering. What you control is which plugins are present and which zone each block owns. Longest-suffix match decides which block handles a query, so `web.lab.local` goes to the first block and `example.com` to the second.

Now the zone file itself, in standard RFC 1035 master-file format - the same format BIND has used for decades:

```bash
sudo tee /etc/coredns/db.lab.local >/dev/null <<'EOF'
$ORIGIN lab.local.
$TTL 60

@       IN  SOA ns.lab.local. admin.lab.local. (
                2026072801  ; serial
                7200        ; refresh
                3600        ; retry
                1209600     ; expire
                60          ; minimum (negative cache TTL)
                )

@       IN  NS      ns.lab.local.

ns      IN  A       10.30.1.53
web     IN  A       10.30.1.53
db      IN  A       10.30.1.20
www     IN  CNAME   web.lab.local.
mail    IN  A       10.30.1.53
@       IN  MX      10 mail.lab.local.
@       IN  TXT     "v=spf1 -all"
_http._tcp IN SRV   10 10 80 web.lab.local.
EOF
```

`$ORIGIN` sets the suffix that bare names get; `@` means the origin itself (the zone apex); `$TTL` sets the default. Note that the apex carries SOA and NS - and therefore, per the rule above, could never be a CNAME. `www` *is* a CNAME, and it points at `web.lab.local.` with a trailing dot, because a name without the trailing dot would get `$ORIGIN` appended to it and you'd end up with `web.lab.local.lab.local.` That missing dot is a common zone file bug.

Run it:

```bash
sudo /usr/local/bin/coredns -conf /etc/coredns/Corefile
```

Leave it in the foreground - the `log` plugin will print every query, which is exactly what we want to watch. From `client`, in another session:

```bash
dig +noall +answer web.lab.local
dig +noall +answer www.lab.local
dig +noall +answer lab.local MX
dig +noall +answer lab.local SOA
dig +noall +answer _http._tcp.lab.local SRV
dig +noall +answer nonexistent.lab.local
```

The `www` lookup returns both the CNAME and the A record it resolves to - CoreDNS chased its own alias. The last one returns nothing in the answer section; check the full output and you'll see `status: NXDOMAIN` and an AUTHORITY section containing the SOA. That SOA is how the resolver learns the negative-caching TTL, from the minimum field we set to 60.

```bash
dig nonexistent.lab.local | grep -E 'status|SOA'
```

Same zone, same answers, completely different implementation. dnsmasq gave us the records as one-line config directives; CoreDNS gave us a real zone file and a plugin chain. In a Kubernetes cluster that same Corefile pattern is there, just with a `kubernetes` plugin in the first block instead of `file`, generating records from the API server on the fly.

## Debugging tools

### dig

The one to reach for. The flags that carry their weight:

```bash
dig example.com                    # full output: header, flags, question, answer, timing
dig +short example.com             # just the data - good for scripts
dig +noall +answer example.com     # the answer section, formatted, nothing else
dig @10.30.1.53 web.lab.local      # ask a specific server, bypassing resolv.conf
dig +trace www.example.com         # walk the hierarchy from the root yourself
dig +norecurse @8.8.8.8 example.com # ask for cache contents only, no chasing
dig -x 10.30.1.53                  # reverse lookup, does the in-addr.arpa dance for you
dig +tcp example.com               # force TCP instead of UDP
dig +nssearch example.com          # query every authoritative NS and compare their SOAs
dig +stats example.com             # query time and which server answered
```

`@server` is the most valuable habit here: when something resolves inconsistently, ask each candidate server directly and compare. That separates "the data is wrong" from "one resolver has a stale copy".

### Reading a dig answer

```
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 47823
;; flags: qr rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1
```

**status** is the first thing to look at:

- `NOERROR` - the query succeeded. Note this does *not* mean you got data. A NOERROR with zero answers means "that name exists, but not with the type you asked for" - a NODATA response. Asking for the AAAA of an IPv4-only host does this.
- `NXDOMAIN` - the name does not exist at all. Authoritative and cacheable (for the SOA minimum).
- `SERVFAIL` - the server tried and failed. Broken upstream, broken delegation, DNSSEC validation failure, or timeouts. This is the ambiguous one, and usually means "go look at the server, not the name".
- `REFUSED` - the server declined to answer. Usually an ACL: you're not allowed to query it, or it isn't a resolver for you.

**flags**:

- `qr` - this is a response, not a query.
- `rd` - recursion desired (you asked for it).
- `ra` - recursion available (the server offers it). A missing `ra` on a server you thought was a resolver means it's authoritative-only.
- `aa` - authoritative answer, straight from a server that owns the zone.
- `tc` - truncated; the answer didn't fit in UDP and you should retry over TCP.
- `ad` - authenticated data, i.e. DNSSEC-validated.

**Sections**: ANSWER is what you asked for. AUTHORITY tells you which servers are authoritative for the zone - and on an NXDOMAIN it holds the SOA that sets negative caching. ADDITIONAL holds records the server volunteered because you'll probably need them, most importantly glue A records for nameservers named in AUTHORITY.

### host and nslookup

`host` is the quick one-liner, `nslookup` is the legacy one you'll find installed everywhere including on machines where nothing else is:

```bash
host web.lab.local
host -t MX gmail.com
host 8.8.4.4

nslookup web.lab.local
nslookup web.lab.local 10.30.1.53
```

`nslookup` is effectively deprecated - its output is less informative and it has historically had quirks around search domains and error reporting. But it is on every Windows box and every minimal container, so it's worth knowing. When you have `dig`, use `dig`.

### drill

From ldns, part of the `ldnsutils` package we installed on `client`. Same job as `dig`, notably better at showing DNSSEC chains:

```bash
drill web.lab.local @10.30.1.53
drill -T www.example.com        # trace, like dig +trace
drill -S example.com            # chase the full DNSSEC signature chain
```

### resolvectl

On systemd hosts (not ours anymore, since we disabled it, but common elsewhere):

```bash
resolvectl status               # per-link DNS servers, search domains, DNSSEC mode
resolvectl query example.com    # resolve through resolved, not by raw DNS packet
resolvectl statistics           # cache hits and misses
resolvectl flush-caches         # clear the local cache
```

`resolvectl query` matters because it goes through the same path your applications do, unlike `dig`. When `dig` and `resolvectl query` disagree, the problem is in `resolved`'s configuration or cache, not in DNS.

### getent

Already mentioned, but it belongs in this list:

```bash
getent hosts web.lab.local
getent ahosts web.lab.local     # shows both address families and socket types
```

This is the ground truth for "what will my application see", because it walks nsswitch exactly like `getaddrinfo()` does.

### tcpdump

When the answers don't make sense, stop believing the tools and watch the wire:

```bash
sudo tcpdump -i any -n port 53
sudo tcpdump -i any -n -s0 port 53 -vv     # decode the DNS payload in detail
```

Remember the stub resolver caveat: on a systemd-resolved host, application queries go to `127.0.0.53` on loopback. `-i any` catches both that and the real upstream traffic; `-i eth0` catches only the latter.

## A deliberate breakage

Let's break something on purpose and work it out with the tools.

Make sure CoreDNS is still running on `dnsserver`. On `client`, change the search domain to something wrong, and change nothing else:

```bash
# on client
sudo tee /etc/resolv.conf >/dev/null <<'EOF'
nameserver 10.30.1.53
search prod.lab.local lab.local
options ndots:2
EOF
```

Now:

```bash
ping -c1 web
```

That still works - you'd expect it to, `lab.local` is still in the search list. Now try this:

```bash
dig +short web
ping -c1 db.lab.local
curl -sI http://web.lab.local | head -1
```

And now the interesting one:

```bash
ping -c1 web.lab.local
```

Look at the timing. Compare it against `ping -c1 web.lab.local.` - with a trailing dot. On the CoreDNS log on `dnsserver`, watch how many queries each of those four commands generates.

`options ndots:2` says: a name with **fewer than 2 dots** is assumed to be relative, so the search list is tried first. `web` has zero dots, so the resolver tries `web.prod.lab.local` first (NXDOMAIN - that zone doesn't exist in our CoreDNS config), and only then `web.lab.local`, which succeeds. Half the queries were wasted, and if `prod.lab.local` had been served by a slow or unreachable server, that waste would have been measured in seconds rather than microseconds.

`web.lab.local` has two dots, which is *not* fewer than two, so it's tried as-is first and succeeds immediately - the search list never comes into it.

One thing to be careful about when counting lines in the log: `ping` and `curl` call `getaddrinfo()`, which requests both A and AAAA, so each name they resolve produces a pair of queries regardless of the search list. The AAAA comes back NODATA (`NOERROR` with zero answers) because our zone has no IPv6 records. `dig +short web` asks only for A, so it's the cleaner instrument for isolating search-list behaviour - compare `dig +short web` against `dig +short web.lab.local.` and you're looking at search-list effects alone.

The trailing dot in `web.lab.local.` makes the name **fully qualified** explicitly - the resolver is forbidden from applying the search list to it at all, regardless of `ndots`. That's a genuinely useful debugging move: if adding a trailing dot fixes or changes the behaviour, your problem is the search list.

Now do the same experiment with a stale cache. On `dnsserver`, edit the zone file to change `web`'s address:

```bash
# on dnsserver - change web to 10.30.1.99 and bump the serial
sudo sed -i -E 's/^(web[[:space:]]+IN[[:space:]]+A[[:space:]]+)10\.30\.1\.53/\110.30.1.99/' /etc/coredns/db.lab.local
sudo sed -i 's/2026072801/2026072802/' /etc/coredns/db.lab.local

# confirm the edit landed on web and nothing else
grep -E '^(ns|web|db)' /etc/coredns/db.lab.local
```

From `client`, immediately:

```bash
dig +short web.lab.local
```

There's a good chance you still get `10.30.1.53`, and the reason is worth pinning down precisely, because there are two quite different mechanisms that could produce it.

The first is the server not having reloaded. CoreDNS's `file` plugin reads the zone at startup and watches the file for changes, but the reload isn't instantaneous and a malformed edit leaves it serving the last version it successfully parsed. Check the CoreDNS log - a reload is reported there, and so is a parse error.

The second is a cache holding the old answer. Note that our Corefile puts the `cache` plugin only on the catch-all `.:53` block, *not* on the `lab.local:53` block, so CoreDNS itself is not caching this zone's answers. That leaves the client end. `dig` doesn't cache at all, so if you're seeing a stale answer through `dig` it's the server side, not the client.

To see client-side caching properly you need something that caches - which is exactly what we removed when we disabled `systemd-resolved`. We made the caching layer visible by taking it out. Put it back and the picture changes:

```bash
# on client
sudo systemctl enable --now systemd-resolved
resolvectl query web.lab.local     # goes through resolved's cache
resolvectl statistics              # watch hits climb on a repeat query
resolvectl flush-caches            # and this is the fix when it's stale
```

Either way, the operational point stands: **the moment you publish a record with a TTL, you have committed to it potentially being live for TTL seconds after you change it** - in some resolver, on some machine, that you do not control and cannot flush. Our TTL is 60, so the worst case is a minute. With a 24-hour TTL and no advance planning, it's a day. Lower the TTL first, wait out the old one, then make the change.

## Wrapping up

Let's recap the shape of what we built.

- **Addresses are what's reachable.** DNS is a naming layer on top; it adds indirection, not connectivity. Every DNS problem should be checked against "does it work by IP?" first - if it doesn't, DNS was never the issue.
- **`/etc/hosts`** is a flat, local, TTL-less, non-delegating file. Useful for pinning and overrides, unworkable as infrastructure.
- **`/etc/nsswitch.conf`** decides the *order* of resolution backends. `files dns` is why hosts beats DNS. Nothing about that is inherent.
- **`dig` bypasses nsswitch and `/etc/hosts` entirely.** It speaks DNS to a server. `getent hosts` is the tool that shows you what your application will actually see. When those two disagree, that disagreement *is* the bug.
- **`/etc/resolv.conf`** holds `nameserver`, `search`, and `options`. On modern Ubuntu it's usually a symlink to a systemd-resolved stub pointing at `127.0.0.53`, which means your queries take a loopback hop through a second cache before they ever reach the network.
- **A recursive resolver** chases answers through the hierarchy on your behalf and caches them. **An authoritative server** holds zone data and answers only for what it owns, returning referrals for what it has delegated. They are different jobs. `dig +trace` shows you the difference in a single command.
- **TTLs** govern how long anyone may hold an answer, and **the SOA minimum** governs how long a *negative* answer sticks. Both are planning tools.
- **CNAMEs** cannot coexist with other records at the same name and therefore cannot live at a zone apex.
- **PTR** records live in `in-addr.arpa` with the octets reversed, are delegated by address-block owners, and need not agree with forward DNS.
- **dnsmasq and CoreDNS** are two implementations of the same ideas - one as terse config directives, one as a plugin chain over a standard zone file. Which one is in front of you changes the syntax, not the concepts.

As with the addressing, almost none of this configuration survives a reboot in the form we wrote it, and that's deliberate - it kept the moving parts visible. In the real world:

- Addresses and routes on Ubuntu go in netplan YAML under `/etc/netplan/`.
- `/etc/resolv.conf` is normally not hand-edited at all; you'd either let `systemd-resolved` manage it and configure DNS per-link (via netplan's `nameservers:` block, or `resolvectl dns`, which is itself non-persistent), or set `DNSStubListener=no` in `/etc/systemd/resolved.conf` if you truly want your own resolver on port 53.
- `dnsmasq`'s config in `/etc/dnsmasq.d/` *is* the persistent form - that part we did properly.
- CoreDNS we ran in the foreground by hand; in production it's a systemd unit (or, far more often, a Deployment in Kubernetes with the Corefile in a ConfigMap).
- Zone data in any real environment lives in version control and is deployed, not `sed`-edited in place. And the serial gets bumped, every time.

To tear it down:

```bash
limactl stop dnsserver
limactl stop client

limactl delete dnsserver
limactl delete client

limactl network delete --force dnsnet
```
