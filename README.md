<div align="center">

# 🔍 ARP: How a Ping Really Starts

### A Frame-by-Frame Walkthrough in Cisco Packet Tracer Simulation Mode

![Cisco](https://img.shields.io/badge/Cisco-Packet%20Tracer-1BA0D7?style=for-the-badge&logo=cisco&logoColor=white)
![ARP](https://img.shields.io/badge/Protocol-ARP-red?style=for-the-badge)
![Layer2](https://img.shields.io/badge/Layer-2%20%2F%203-orange?style=for-the-badge)
![Switching](https://img.shields.io/badge/Topic-Switching-purple?style=for-the-badge)
![Mode](https://img.shields.io/badge/Mode-Simulation-yellow?style=for-the-badge)
![Level](https://img.shields.io/badge/Level-CCNA-green?style=for-the-badge)
![Status](https://img.shields.io/badge/Status-Verified-success?style=for-the-badge)

*IP tells you where to go. MAC tells the switch how to get there. ARP connects the two.*

</div>

---

## 📑 Table of Contents
1. [Overview](#-overview)
2. [Topology](#-topology)
3. [Addressing Table](#-addressing-table)
4. [ARP Theory in 60 Seconds](#-arp-theory-in-60-seconds)
5. [The Scenario](#-the-scenario)
6. [Stage-by-Stage Walkthrough](#-stage-by-stage-walkthrough)
7. [Frame Summary](#-frame-summary)
8. [Switch Decisions](#-switch-decisions)
9. [Why the Router Drops the ARP Request](#-why-the-router-drops-the-arp-request)
10. [Final MAC Address Table](#-final-mac-address-table)
11. [Key Takeaways](#-key-takeaways)
12. [Troubleshooting Notes](#-troubleshooting-notes)
13. [Next Labs](#-next-labs)
14. [Author](#-author)

---

## 🎯 Overview

Before a device can send a frame to another device on the same network, it needs the destination's **MAC address**. **ARP (Address Resolution Protocol)** finds it.

In this lab I used **Packet Tracer Simulation mode** to follow a single `ping` from PC0 to PC1 and inspected the PDU at every device. The goal is to see **who sends what, to whom, and why**.

| Item | Detail |
|---|---|
| Protocol studied | ARP (plus ICMP echo as the trigger) |
| Tool | Cisco Packet Tracer, Simulation mode |
| Test | `ping` from PC0 to PC1 (same subnet) |
| Starting state | PC0 does not know PC1's MAC, and Switch0 does not know PC1's port |
| Devices inspected | PC0, Switch0, Router0, PC1 |

---

## 🗺 Topology

```
                  Router0 (2901)
        192.168.1.1/24 ●━━━●  200.100.100.1/24
              (Gi0/0)  │         │
                 Fa0/24│         │
               ┌───────┴──┐   ┌──┴────────┐
               │ Switch0  │   │ Switch1   │
               │ 2950-24  │   │ 2950-24   │
               └──┬────┬──┘   └──┬─────┬──┘
             Fa0/1│    │Fa0/2    │     │
                  │    │         │     │
                PC0   PC1       PC2   PC3
          192.168.1.2 192.168.1.3  200.100.100.2 200.100.100.3
```

**Test: PC0 → PC1 ping (Process 1).** Switch0 is the only switch involved.

---

## 📋 Addressing Table

| Device | Interface / Port | IP Address | MAC Address |
|---|---|---|---|
| **PC0** | FastEthernet0 (Switch0 Fa0/1) | 192.168.1.2/24 | 0005.5EA8.D53D |
| **PC1** | FastEthernet0 (Switch0 Fa0/2) | 192.168.1.3/24 | 0004.9A33.87CC |
| Switch0 | (2950-24) | none | 00E0.F93E.17C2 |
| Router0 (2901) | Gi0/0 (Switch0 Fa0/24) | 192.168.1.1/24 | 0060.3E9E.2801 (as learned on Fa0/24) |
| Router0 (2901) | Far side | 200.100.100.1/24 | n/a |
| Switch1 | (2950-24) | none | 00D0.FF8D.2E18 |
| PC2 | FastEthernet0 | 200.100.100.2/24 | 000C.CF52.A56A |
| PC3 | FastEthernet0 | 200.100.100.3/24 | 0002.4AC2.2299 |

---

## 📚 ARP Theory in 60 Seconds

| Concept | Meaning |
|---|---|
| ARP | Maps a known **IP address** to an unknown **MAC address** on the local network |
| ARP Request | "Who has IP x.x.x.x? Tell me your MAC." Sent as a **broadcast** (`FFFF.FFFF.FFFF`) |
| ARP Reply | "I have it, my MAC is y." Sent as a **unicast** straight back to the requester |
| ARP cache | Table on each PC of IP → MAC mappings it has learned |
| MAC address table | Table on a switch of MAC → port mappings, learned from **source MACs** |
| Flooding | A switch sends broadcast (and unknown unicast) frames out **all ports except the incoming port** |
| Broadcast domain | The area a broadcast can reach. **Routers stop it.** |

### Same subnet vs different subnet

| Destination | The PC ARPs for... |
|---|---|
| Same subnet (this lab) | The **destination itself** |
| Different subnet | The **default gateway** (the router) |

---

## 🎬 The Scenario

```
 PC0 wants to ping PC1.

 PC0 knows:   PC1's IP address (192.168.1.3)
 PC0 needs:   PC1's MAC address
 Switch0:     has no entry for PC1's MAC yet
```

```
   PC0 ──(1) ARP request  [broadcast]──────► Switch0 ──flood──► PC1
                                                         └─────► Router0 ✖ drops
   PC0 ◄─(2) ARP reply    [unicast]───────── Switch0 ◄────────── PC1
   PC0 ──(3) ICMP request [unicast]────────► Switch0 ──────────► PC1
   PC0 ◄─(4) ICMP reply   [unicast]───────── Switch0 ◄────────── PC1
```

---

## 🔬 Stage-by-Stage Walkthrough

### Stage 1: PC0 creates the ICMP echo request

```
At Device: PC0        Source: PC0        Destination: PC1
Layer 3: IP Header   Src IP: 192.168.1.2   Dest IP: 192.168.1.3
         ICMP Message Type: 8 (Echo Request)
Layer 2: (empty)
```

**Annotations**
- The ping process builds the ICMP echo request and sets the TTL.
- Packet Tracer notes that the destination is in the **same subnet**, so the next hop is the **destination itself**.
- **Layer 2 is blank.** PC0 has no MAC for 192.168.1.3 yet, so it cannot build the Ethernet frame. This triggers **ARP**.

---

### Stage 2: PC0 builds the ARP request (broadcast)

```
At Device: PC0        Source: PC0        Destination: Broadcast
Layer 2: Ethernet II Header
         0005.5EA8.D53D  >>  FFFF.FFFF.FFFF
         ARP Packet   Src IP: 192.168.1.2   Dest IP: 192.168.1.3
Layer 1: Port FastEthernet0
```

**Annotations**
- The ARP process builds a request for the target IP and encapsulates it in an Ethernet frame.
- **Destination MAC = `FFFF.FFFF.FFFF`**: the Layer 2 broadcast address, so **every** device on the segment receives it.
- The frame carries PC0's own MAC, so the answer knows where to return.

---

### Stage 3: Switch0 receives the ARP request

```
At Device: Switch0
In  Layers: Layer 2: 0005.5EA8.D53D >> FFFF.FFFF.FFFF   Port FastEthernet0/1
Out Layers: Layer 2: 0005.5EA8.D53D >> FFFF.FFFF.FFFF   Ports FastEthernet0/2, FastEthernet0/24
```

**Annotations**
- The frame arrives on **Fa0/1** (PC0). The switch learns **PC0's MAC → Fa0/1**.
- The destination is the broadcast address, so the switch processes it as a broadcast.
- Packet Tracer notes that the switch's own VLAN interface is not up, so the switch **does not answer** the ARP itself. It only forwards.

---

### Stage 4: Switch0 floods the broadcast

```
"This is a broadcast frame. The Switch sends out the frame to all ports
 in the same VLAN except the receiving port."

Out ports: FastEthernet0/2  (PC1)
           FastEthernet0/24 (Router0)
```

**Annotation:** The switch can't know where 192.168.1.3 lives, so it copies the frame to **every other port**. Fa0/1 is excluded because that is where it came from.

---

### Stage 5: Router0 receives and drops the request

```
At Device: Router0    In Layer: Layer 2: 0005.5EA8.D53D >> FFFF.FFFF.FFFF
                                ARP Src IP: 192.168.1.2  Dest IP: 192.168.1.3
                      Layer 1: Port GigabitEthernet0/0

 1. Destination MAC matches broadcast
 2. Decapsulates the frame
 3. It is an ARP request
 4. Target IP does NOT match the receiving port's IP
 5. Proxy ARP is enabled on the receiving port
 6. Checks the routing table: is the requested IP reachable?
 7. Yes, it is reachable on the RECEIVING port
 8. The ARP process drops the frame
```

**Annotation:** The router rejects the request. Full explanation in [Why the Router Drops the ARP Request](#-why-the-router-drops-the-arp-request).

---

### Stage 6: PC1 answers with the ARP reply (unicast)

```
At Device: PC1
In  Layers: Layer 2: 0005.5EA8.D53D >> FFFF.FFFF.FFFF
            ARP Src IP: 192.168.1.2   Dest IP: 192.168.1.3   Port FastEthernet0
Out Layers: Layer 2: 0004.9A33.87CC >> 0005.5EA8.D53D
            ARP Src IP: 192.168.1.3   Dest IP: 192.168.1.2   Port FastEthernet0
```

**Annotations**
- PC1 sees that the **target IP is its own**, so it replies using its receiving port's MAC address.
- The reply is **unicast**: destination `0005.5EA8.D53D` (PC0). PC1 got PC0's MAC from the request itself.
- Roles reverse: **PC1 is now the source and PC0 the destination.**

---

### Stage 7: Switch0 delivers the ARP reply

```
At Device: Switch0
In  Layers: Layer 2: 0004.9A33.87CC >> 0005.5EA8.D53D   Port FastEthernet0/2
Out Layers: Layer 2: 0004.9A33.87CC >> 0005.5EA8.D53D   Port FastEthernet0/1
```

**Annotations**
- The reply arrives on **Fa0/2**, so the switch learns **PC1's MAC → Fa0/2**.
- It already knows PC0's MAC is on **Fa0/1**, so the frame is forwarded **only** out Fa0/1. **No flooding this time.**
- PC0 now has PC1's MAC. The resolution is complete.

---

### Stage 8: PC0 sends the ICMP echo request (unicast)

```
At Device: Switch0    Source: PC0    Destination: PC1
In  Layers: Layer 2: 0005.5EA8.D53D >> 0004.9A33.87CC   Port FastEthernet0/1
Out Layers: Layer 2: 0005.5EA8.D53D >> 0004.9A33.87CC   Port FastEthernet0/2
```

**Annotation:** With the MAC resolved, the original ping is released. It now has a proper Layer 2 header and goes out **only to Fa0/2**.

---

### Stage 9: PC1 receives the request and replies

```
At Device: PC1
In  Layers: Layer 3: Src IP 192.168.1.2  Dest IP 192.168.1.3   ICMP Type 8
            Layer 2: 0005.5EA8.D53D >> 0004.9A33.87CC          Port FastEthernet0
Out Layers: Layer 3: Src IP 192.168.1.3  Dest IP 192.168.1.2   ICMP Type 0
            Layer 2: 0004.9A33.87CC >> 0005.5EA8.D53D          Port FastEthernet0
```

**Annotations**
- The destination MAC matches PC1's port, so PC1 decapsulates the frame.
- **ICMP Type 8 = Echo Request** in, **Type 0 = Echo Reply** out.

---

### Stage 10: PC0 receives the echo reply: ping succeeds ✅

```
At Device: PC0
In Layers: Layer 3: Src IP 192.168.1.3  Dest IP 192.168.1.2   ICMP Type 0
           Layer 2: 0004.9A33.87CC >> 0005.5EA8.D53D          Port FastEthernet0

 1. Destination IP matches the device's IP: de-encapsulates
 2. The ICMP process received an Echo Reply
 3. The Ping process received an Echo Reply message
```

**Annotation:** The full round trip is complete. The ARP exchange was needed only once, and later pings reuse the cached entries.

---

## 🧾 Frame Summary

| # | Frame | Src MAC | Dst MAC | Src IP | Dst IP | Cast |
|---|---|---|---|---|---|---|
| 1 | **ARP Request** | 0005.5EA8.D53D | `FFFF.FFFF.FFFF` | 192.168.1.2 | 192.168.1.3 | **Broadcast** |
| 2 | **ARP Reply** | 0004.9A33.87CC | 0005.5EA8.D53D | 192.168.1.3 | 192.168.1.2 | **Unicast** |
| 3 | ICMP Echo Request (Type 8) | 0005.5EA8.D53D | 0004.9A33.87CC | 192.168.1.2 | 192.168.1.3 | Unicast |
| 4 | ICMP Echo Reply (Type 0) | 0004.9A33.87CC | 0005.5EA8.D53D | 192.168.1.3 | 192.168.1.2 | Unicast |

Frames 3 and 4 only became possible **after** the ARP exchange.

---

## 🔀 Switch Decisions

| Frame | Arrives on | Does the switch know the destination? | Action | Leaves on |
|---|---|---|---|---|
| ARP Request | Fa0/1 | Broadcast | **Flood** | Fa0/2, Fa0/24 |
| ARP Reply | Fa0/2 | Yes (PC0 on Fa0/1) | Forward | Fa0/1 only |
| ICMP Echo Request | Fa0/1 | Yes (PC1 on Fa0/2) | Forward | Fa0/2 only |
| ICMP Echo Reply | Fa0/2 | Yes (PC0 on Fa0/1) | Forward | Fa0/1 only |

> The last row follows the simulation event list. The PDU details were captured for the first three.

---

## 🚧 Why the Router Drops the ARP Request

Router0 received the broadcast on Gi0/0 and chose to drop it. Packet Tracer shows the reasoning:

| Check | Result |
|---|---|
| Is the target IP the router's own address? | ❌ No |
| Is Proxy ARP enabled on this port? | ✅ Yes |
| Is the requested IP reachable per the routing table? | ✅ Yes |
| Is it reachable via a **different** interface? | ❌ No, it is on the **receiving** interface |
| **Decision** | **Drop**: the router has nothing to answer or relay |

- Proxy ARP only helps when the target is on **another** network that the router can reach.
- Here, 192.168.1.3 is on the **same** network as the arriving broadcast. PC1 can answer directly, and it does.
- Broadcasts are **never forwarded** by a router, which makes it the boundary of the broadcast domain. This is why ARP only works within a subnet.

---

## 📊 Final MAC Address Table

```
Switch# show mac-address-table

          Mac Address Table
-------------------------------------------

Vlan    Mac Address       Type        Ports
----    -----------       --------    -----
   1    0004.9a33.87cc    DYNAMIC     Fa0/2
   1    0005.5ea8.d53d    DYNAMIC     Fa0/1
   1    0060.3e9e.2801    DYNAMIC     Fa0/24
```

| MAC | Port | Device |
|---|---|---|
| 0005.5ea8.d53d | Fa0/1 | PC0 |
| 0004.9a33.87cc | Fa0/2 | PC1 |
| 0060.3e9e.2801 | Fa0/24 | Router-facing uplink |

**Annotations**
- All three entries are **DYNAMIC**, which means the switch **learned them automatically** from source MACs.
- This table is why Stages 7 to 10 were **unicast**: the switch knows exactly which port leads to each device.
- Dynamic entries age out if the device stays silent.

### Optional extra check: PC0's ARP cache

```
C:\> arp -a
  Internet Address      Physical Address      Type
  192.168.1.3           0004.9a33.87cc        dynamic
```
> 📸 Expected output. Add your own screenshot if you run it.

---

## 💡 Key Takeaways

| # | Takeaway |
|---|---|
| 1 | ARP turns a **known IP** into an **unknown MAC** on the local network |
| 2 | The ARP **request is a broadcast** (`FFFF.FFFF.FFFF`), and the **reply is a unicast** |
| 3 | A switch **learns source MACs** and **floods** broadcasts out every port except the incoming one |
| 4 | After learning, the switch forwards unicast frames to **one port only** |
| 5 | A **router does not forward broadcasts**, so ARP stays inside one subnet |
| 6 | Same subnet: ARP for the **destination**. Different subnet: ARP for the **gateway** |
| 7 | The first packet waits (Layer 2 blank) until ARP resolves the MAC |
| 8 | Simulation mode makes the invisible visible: open each PDU and read the layers |

---

## 🛠 Troubleshooting Notes

| Symptom | Check |
|---|---|
| First ping slow or drops | The first packet waits for ARP to finish, which is normal |
| Ping fails inside the same subnet | Wrong IP or mask? Switch port down? Different VLANs? |
| `arp -a` shows nothing | No successful resolution yet: ping first, then check |
| Wrong MAC in the ARP cache | Duplicate IP on the network, or a stale entry (`arp -d` to clear) |
| Frames flooded all the time | Is the destination silent, or the MAC entry aged out? |
| Cannot reach another subnet | The PC must ARP for the **gateway**, so check the default gateway setting |

---

## 🚀 Next Labs

- [x] ARP within one subnet (Process 1: PC0 → PC1) ✅
- [ ] ARP across subnets (Process 2: PC0 → PC2, ARP for the gateway)
- [ ] Proxy ARP in action
- [ ] MAC address table aging and static MAC entries
- [ ] ARP security: spoofing and Dynamic ARP Inspection

---

## 👨‍💻 Author

**Maaz Khan**
CCNA-certified Network & NOC Engineer · Lower Dir, KPK, Pakistan

[![LinkedIn](https://img.shields.io/badge/LinkedIn-maazkhanms-0A66C2?style=for-the-badge&logo=linkedin)](https://www.linkedin.com/in/maazkhanms)
[![GitHub](https://img.shields.io/badge/GitHub-maazkhanms-181717?style=for-the-badge&logo=github)](https://github.com/maazkhanms)

⭐ If this lab helped you, give the repo a star!
