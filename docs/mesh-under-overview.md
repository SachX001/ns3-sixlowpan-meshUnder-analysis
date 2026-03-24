# Mesh-Under Routing in ns-3 6LoWPAN

## Overview
Mesh-under routing in 6LoWPAN is a forwarding mechanism that operates at the adaptation layer between the IEEE 802.15.4 MAC layer and the IPv6 network layer. Instead of performing routing decisions at the IP layer (route-over routing), mesh-under routing forwards packets using Layer-2 broadcast propagation combined with hop-limited forwarding.

In ns-3, mesh-under routing is implemented inside the SixLowPanNetDevice and is primarily handled within the function:

$${\\color{red}SixLowPanNetDevice::ReceiveFromDevice()}$$   

This function is responsible for:
- parsing dispatch headers
- extracting mesh and broadcast headers
- detecting duplicate packets
- enforcing hop limits
- scheduling packet rebroadcast using jitter

Mesh-under routing uses controlled flooding to propagate packets across the network.

---

## Mesh-Under vs Route-Over Routing
There are two major forwarding strategies defined for 6LoWPAN networks:

### Route-over routing
Routing decisions occur at the IPv6 layer.

Characteristics:
- each node acts as an IPv6 router
- packets are forwarded based on IP routing tables
- fragmentation and reassembly occur at each hop
- higher memory and processing requirements

### Mesh-under routing
Routing decisions occur below the IPv6 layer.

Characteristics:
- forwarding handled by the adaptation layer
- packets remain fragmented during forwarding
- lower processing overhead
- suitable for low-power wireless mesh networks

ns-3 currently implements mesh-under forwarding using broadcast-based propagation with duplicate suppression.

---

## Mesh Header (LOWPAN_MESH)
Mesh-under routing uses the LOWPAN_MESH header to support multi-hop forwarding.

The mesh header contains:
- originator address
- final destination address
- hops left field

These fields allow intermediate nodes to:
- identify the source node
- determine whether they are the final destination
- control forwarding lifetime using hop limits

The hops-left field functions similarly to a TTL (Time-To-Live) value and prevents infinite packet propagation.

---

## Broadcast Header (LOWPAN_BC0)
Broadcast packets include an additional BC0 header.

The BC0 header contains:
- broadcast sequence number
 This sequence number is used together with the originator address to uniquely identify broadcast packets:
 (originator address, sequence number)
 Intermediate nodes store these identifiers inside a duplicate detection cache:
 $${\\color{red}m_seenPkts}$$   
 This mechanism prevents rebroadcast loops and ensures controlled flooding behavior.

---

## Duplicate Detection Mechanism
Duplicate detection in ns-3 mesh-under forwarding is implemented using:
$${\\color{red}std::map<Address, std::list<uint8_t>> m_ seenPkts}$$   
Each node stores sequence numbers received from each originator node.

When a packet arrives:
   ```
  if packet already exists in cache:
    drop packet

  else:
    store packet identity
    continue forwarding logic
  ```

This ensures that each node forwards a broadcast packet at most once.

---

## Rebroadcast Scheduling with Jitter
When a node receives a packet for the first time and determines that forwarding is required, it schedules a rebroadcast operation after a randomized delay:
```
Simulator::Schedule(
    MilliSeconds(m_meshUnderJitter->GetValue()),
    ...
)
```
The purpose of jitter is to:
- reduce transmission collisions
- avoid simultaneous rebroadcast storms
- distribute forwarding events across time

However, the current implementation schedules rebroadcast immediately after first reception without considering neighborhood forwarding activity.

---

## Limitation of Current Implementation
The existing forwarding strategy follows this pipeline:
receive packet
→ check duplicate cache
→ decrement hop limit
→ schedule rebroadcast after jitter

While this guarantees reliable propagation, it may produce redundant transmissions in dense mesh deployments because nodes rebroadcast packets even when neighboring nodes are already forwarding them.
This motivates the introduction of adaptive rebroadcast suppression strategies based on duplicate arrival observations during the jitter interval.

---

## Target for Improvement
The improvement proposed in this repository focuses on enhancing mesh-under forwarding efficiency inside:
$${\\color{red}SixLowPanNetDevice::ReceiveFromDevice()}$$   

Specifically, the rebroadcast scheduling stage can be extended to:
- observe duplicate packet arrivals during jitter window
- estimate neighborhood forwarding activity
- suppress redundant rebroadcasts when sufficient coverage is detected

This enables controlled flooding behavior while preserving delivery reliability.
