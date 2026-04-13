# Lessons Learned & Troubleshooting Journal

> Real problems encountered during the PoC build, what caused them, and how they were resolved. This is the stuff that doesn't make it into the polished report but matters more than anything for actually learning.

---

## 1. Packet Tracer CLI `ipconfig` Does Not Persist

### The Problem
After building the full topology, configuring VLANs on the switch, setting up router-on-a-stick with sub-interfaces, and getting everything looking perfect on the infrastructure side — **no end device could communicate with anything**.

Pings from Clinical-PC to the EHR-Server timed out. Pings between all VLANs failed. HTTP was dead. The router sub-interfaces were up/up, the trunk was passing all VLANs, the switch port assignments were correct. Everything about the infrastructure was right.

### The Root Cause
I configured the end device IP addresses using the **command-line `ipconfig` command** inside the PC's Command Prompt:

```
C:\> ipconfig 192.168.10.10 255.255.255.0 192.168.10.1
```

The command appeared to work — running `ipconfig /all` showed the correct IP, subnet mask, and gateway. But Packet Tracer's simulated PC **did not actually apply the configuration to the network interface**. The device was sending traffic with no valid IP.

### The Fix
Configure static IPs through the **GUI only**: click the device → **Desktop** tab → **IP Configuration** → select **Static** → manually enter IP, subnet, and gateway.

### The Lesson
Packet Tracer's PC simulation is not a real operating system. The CLI `ipconfig` command is cosmetic in some versions — it updates the display but doesn't always bind the address to the simulated NIC. **Always use the GUI for end device IP configuration in Packet Tracer.** This cost me over an hour of debugging infrastructure that was never broken.

---

## 2. The `log` Keyword Is Not Supported in Packet Tracer ACLs

### The Problem
When entering extended ACL deny statements with the `log` keyword:

```
deny ip 192.168.30.0 0.0.0.255 192.168.10.0 0.0.0.255 log
```

Packet Tracer did not accept it as expected. After the deny/permit and destination, Packet Tracer's CLI only offers `dscp` or `precedence` as options — there is no `log` keyword available.

### The Root Cause
Packet Tracer runs a **simplified subset of Cisco IOS**. Many real IOS features are not implemented, including:
- The `log` keyword on ACL entries
- Named ACL sequence number insertion in some versions
- Various show command options

### The Fix
Remove the `log` keyword from all ACL entries. Use `show ip access-lists` to view **match counters** instead — this provides equivalent verification that rules are matching traffic, just without per-packet syslog output.

### The Lesson
When building labs in Packet Tracer, **always test commands as you go** rather than scripting everything from documentation and pasting it in. Real IOS documentation doesn't apply 1:1 to Packet Tracer. The match counters on `show ip access-lists` turned out to be better evidence for the report anyway — a single screenshot shows cumulative proof that the ACLs are working.

---

## 3. ACL Order of Operations — Specific Permits Before General Denies

### The Problem
The CLINICAL_ACL needed to:
1. **Permit** TCP 80/443 from VLAN 10 to the EHR-Server (192.168.40.10)
2. **Deny** all other traffic from VLAN 10 to VLAN 40
3. **Deny** traffic from VLAN 10 to VLANs 20 and 30

Initially, HTTP from Clinical-PC to the EHR-Server was timing out even though the permit rules were in place.

### The Root Cause
Extended ACLs in Cisco IOS are processed **top-down, first match wins**. The specific TCP permit entries (lines 10 and 20) must appear **before** the blanket deny for the VLAN 40 subnet (line 50). If the deny had been placed first, it would have matched all traffic to 192.168.40.0/24 and dropped the HTTP packets before the permit lines were ever evaluated.

The working order:
```
10 permit tcp 192.168.10.0 0.0.0.255 host 192.168.40.10 eq www
20 permit tcp 192.168.10.0 0.0.0.255 host 192.168.40.10 eq 443
30 deny   ip 192.168.10.0 0.0.0.255 192.168.20.0 0.0.0.255
40 deny   ip 192.168.10.0 0.0.0.255 192.168.30.0 0.0.0.255
45 deny   ip 192.168.10.0 0.0.0.255 192.168.40.0 0.0.0.255
60 permit ip 192.168.10.0 0.0.0.255 any
```

### The Lesson
ACL design is about **precision ordering**. The concept of "most specific first, most general last" is taught in every networking course, but actually debugging it when HTTP silently times out and you can't tell if it's the ACL, the server, or the network — that's where the real understanding comes from. This is directly relevant to the CCNA exam where ACL ordering is a core topic.

---

## 4. Trunk Port Configuration — Don't Assume Defaults

### The Problem
After configuring VLANs and assigning access ports, inter-VLAN routing wasn't working even though the router sub-interfaces were configured correctly.

### The Root Cause
The uplink from the switch to the router needs to be explicitly configured as a **trunk port** carrying all relevant VLANs:

```
interface GigabitEthernet0/1
 switchport mode trunk
 switchport trunk allowed vlan 10,20,30,40
```

Without `switchport mode trunk`, the port defaults to access mode and only carries a single VLAN. The router sub-interfaces receive no tagged frames from the other VLANs.

### The Lesson
Never assume a port is trunking just because you connected it to a router. **Explicitly set trunk mode and specify allowed VLANs.** In production environments, limiting trunk VLANs to only what's needed is also a security best practice — it prevents VLAN hopping attacks by not carrying unnecessary VLANs across the trunk.

---

## 5. Router-on-a-Stick — The Physical Interface Needs `no shutdown`

### The Problem
All four sub-interfaces (Gig0/0.10, .20, .30, .40) showed "up/up" in `show ip interface brief`, but no traffic was flowing.

### The Root Cause
The **parent physical interface** (`GigabitEthernet0/0`) must be explicitly brought up with `no shutdown` before any sub-interfaces will pass traffic. Sub-interfaces inherit the physical state of the parent — if the parent is administratively down, all sub-interfaces are effectively dead even if they show "up" in some status outputs.

### The Fix
```
interface GigabitEthernet0/0
 no shutdown
```

### The Lesson
This is a classic router-on-a-stick gotcha. The sub-interfaces are logical constructs riding on a single physical link. **Always bring up the parent interface first**, then configure sub-interfaces. In my case, this was caught early, but it's worth documenting because it's the kind of thing that wastes 20 minutes if you forget it.

---

## 6. Testing Methodology — Always Establish a Baseline

### The Problem
I initially applied the ACLs immediately after building the topology and then tried to test. When things didn't work, I couldn't tell if the failure was caused by the ACLs doing their job, or by a configuration error in the underlying network.

### The Fix
**Test in two phases:**

1. **Phase 1 (Baseline):** Build the full topology with VLANs and routing but **no ACLs**. Verify that all devices can reach all other devices. Take "before" screenshots. This proves the underlying network works.

2. **Phase 2 (Control):** Apply ACLs. Run the same tests. Failures now are definitively caused by the ACLs, not by misconfigurations. Take "after" screenshots.

### The Lesson
This is basic scientific method applied to security testing — **establish a control before introducing the variable**. It's also how real penetration tests work: you validate the environment first, then test the controls. Skipping the baseline means you can't distinguish between "the ACL blocked it" and "the network was never working."

---

## Key Takeaways

1. **Packet Tracer is a learning tool, not a production simulator.** It has quirks and limitations that real IOS doesn't have. Know the tool's boundaries.

2. **Debug systematically, layer by layer.** When something doesn't work: check physical (cables, interfaces up), then Layer 2 (VLANs, trunking), then Layer 3 (IP addressing, routing), then security (ACLs). Don't jump to blaming ACLs when the problem is a missing IP address.

3. **ACL order matters.** First match wins. Specific permits before general denies. This isn't just exam knowledge — it's the difference between a working security policy and one that breaks clinical access.

4. **Document everything, including failures.** The troubleshooting process taught me more about how ACLs and VLANs actually work than any textbook explanation. The failures are where the learning happens.

5. **Always use the GUI for end device configuration in Packet Tracer.** Seriously. Just do it.
