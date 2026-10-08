# Milestone 3: static route (then remove it)

Goal: fix the missing return path from milestone 2 with a hand-typed route,
see the path with traceroute, then remove the route so OSPF starts clean.

## The problem

Kali → CHR1 (`10.0.12.1`) timed out. Kali's route `10.0.0.0/8 via
192.168.20.1` gets the request to CHR2, and CHR2 is directly on CHR1's subnet,
so the request arrives. But CHR1 only has its connected route
`10.0.12.0/30` and no route back to `192.168.20.0/24`, so it drops the reply.

## What a static route is

A route typed in by an admin. Simple and predictable, but it never changes by
itself: if a link breaks, it keeps pointing at the dead path. That's why ISPs
use routing protocols like OSPF (milestone 4).

## Step 1: add the return route on CHR1

```
/ip route add dst-address=192.168.20.0/24 gateway=10.0.12.2
/ip route print
```

```
DAc  10.0.12.0/30     ether1     distance 0   connected
As   192.168.20.0/24  10.0.12.2  distance 1   static
```

**Distance** = how much the router trusts a route; lower wins when two routes
for the same prefix come from different sources. Connected 0, static 1,
OSPF 110.

## Step 2: test from Kali

```
ping -c 3 10.0.12.1
```

3/3 received, `ttl=63`. CHR1 sends replies with TTL 64; every router that
forwards a packet subtracts 1, so 63 means one router (CHR2) in between. TTL
also stops packets from looping forever.

## Step 3: traceroute

```
traceroute -n 10.0.12.1
```

```
 1  192.168.20.1   CHR2
 2  10.0.12.1      CHR1 (destination)
```

- The sender is not a hop. Each router appears once.
- A router answers from the port facing you: CHR2 shows as `192.168.20.1`,
  not `10.0.12.2`.
- The three times per line are three probes, not three hops.
- Troubleshooting: look at the last hop that answers; the fault is usually
  just past it.

## Step 4: remove the route

```
/ip route remove [find dst-address=192.168.20.0/24]
/ip route print
```

Remove by **what** it is (`[find …]`), not by row number: numbers change when
the table changes, and on a busy router `remove 0` can delete the wrong route.

Kali → CHR1 is back to 100% loss: the request still arrives, the reply has no
way home. The static route is gone so it can't hide whether OSPF works
(static distance 1 would beat OSPF's 110).

Small slips I made:

- `ip route print` on Kali: that's RouterOS syntax. On Linux it's `ip route`.
- `ping -c 10.0.12.1` → `invalid argument`: `-c` needs a count first
  (`-c 3`).

## Milestone 3 complete ✅
