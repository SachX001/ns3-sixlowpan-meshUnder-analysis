# ReceiveFromDevice Forwarding Pipeline in ns-3 6LoWPAN

## Overview

Mesh-under forwarding in ns-3 is implemented inside:

SixLowPanNetDevice::ReceiveFromDevice()

This function is responsible for processing incoming 6LoWPAN packets at the adaptation layer and determining whether packets should:

- be dropped
- be forwarded
- be decompressed
- be delivered to upper layers

This document explains the forwarding pipeline step-by-step.

---

## High-Level Processing Flow

The mesh-under forwarding pipeline follows this sequence:

Incoming packet
→ packet copy created
→ dispatch header parsed
→ mesh header extracted (if present)
→ BC0 header extracted (if present)
→ duplicate detection performed
→ hop limit checked
→ forwarding decision evaluated
→ rebroadcast scheduled with jitter
→ packet delivered upward if destination reached

---

## Step 1: Packet Copy Creation

When a packet arrives, a safe working copy is created:

Ptr<Packet> copyPkt = packet->Copy();

This allows header parsing and manipulation without modifying the original packet buffer.

The copy is used for:

- dispatch parsing
- header extraction
- forwarding decisions
- decompression processing

---

## Step 2: Dispatch Header Parsing

The dispatch byte identifies the type of the next header:

copyPkt->CopyData(&dispatchRawVal, sizeof(dispatchRawVal));

Dispatch types include:

LOWPAN_MESH
LOWPAN_BC0
LOWPAN_FRAG1
LOWPAN_FRAGN
LOWPAN_IPHC
LOWPAN_IPV6

The dispatch byte determines how the packet will be processed next.

---

## Step 3: Mesh Header Extraction

If the packet contains a mesh header:

if (dispatchVal == LOWPAN_MESH)

then:

copyPkt->RemoveHeader(meshHdr);

The mesh header provides:

originator address  
final destination address  
hop limit (hopsLeft)

This information enables multi-hop forwarding decisions at the adaptation layer.

---

## Step 4: Broadcast Header Extraction (BC0)

If the packet contains a BC0 header:

if (dispatchVal == LOWPAN_BC0)

then:

copyPkt->RemoveHeader(bc0Hdr);

The BC0 header provides:

broadcast sequence number

Together with the originator address, this sequence number uniquely identifies broadcast packets:

(originator address, sequence number)

This identifier enables duplicate detection.

---

## Step 5: Duplicate Detection

Duplicate detection uses the structure:

std::map<Address, std::list<uint8_t>> m_seenPkts

Each node maintains a list of sequence numbers received from each originator.

Duplicate detection logic:

if packet already exists in cache:
    drop packet

else:
    store packet identity
    continue forwarding logic

This ensures that each node forwards a broadcast packet at most once.

---

## Step 6: Duplicate Cache Update

If the packet is new:

m_seenPkts[meshHdr.GetOriginator()].push_back(
    bc0Hdr.GetSequenceNumber()
);

This stores the packet identity inside the duplicate detection cache.

To prevent unlimited memory growth:

if cache size exceeds limit:
    remove oldest entry

This keeps the duplicate cache bounded.

---

## Step 7: Forwarding Eligibility Check

Before forwarding, the node verifies whether the packet should be forwarded:

if packet is not addressed to this node
OR packet is broadcast
OR packet is multicast

then forwarding logic continues.

Otherwise:

packet is delivered locally.

---

## Step 8: Hop Limit Enforcement

The mesh header includes a hop limit field:

meshHdr.GetHopsLeft()

If:

hopsLeft == 0

then:

packet is not forwarded further

This prevents infinite propagation loops.

The hop limit functions similarly to a TTL field in IP routing.

---

## Step 9: Originator Protection Check

Nodes must not forward their own packets again:

if meshHdr.GetOriginator() == myAddress

then:

packet is not forwarded

This prevents rebroadcast loops returning to the sender.

---

## Step 10: Rebroadcast Scheduling Using Jitter

If forwarding conditions are satisfied:

meshHdr.SetHopsLeft(hopsLeft - 1);

The packet is scheduled for rebroadcast:

Simulator::Schedule(
    MilliSeconds(m_meshUnderJitter->GetValue()),
    &NetDevice::Send,
    ...
)

The jitter delay helps:

reduce collisions  
avoid synchronized transmissions  
distribute broadcast load across time  

However, the rebroadcast decision is currently made immediately after first reception.

The node does not consider whether neighboring nodes are already forwarding the packet.

---

## Step 11: Fragment Handling

If the packet contains fragmentation headers:

LOWPAN_FRAG1
LOWPAN_FRAGN

then:

ProcessFragment()

is called.

Fragmented packets are reassembled before further processing.

---

## Step 12: Header Decompression

If compression headers are detected:

LOWPAN_IPHC
LOWPAN_HC1

then decompression is performed:

DecompressLowPanIphc()

This restores the IPv6 header before delivery to upper layers.

---

## Step 13: Packet Delivery to Upper Layers

After processing is complete:

m_rxCallback(...)

delivers the packet to the IPv6 layer.

This completes adaptation-layer forwarding.

---

## Forwarding Pipeline Summary

The complete forwarding pipeline:

receive packet
→ parse dispatch header
→ extract mesh header
→ extract BC0 header
→ perform duplicate detection
→ update duplicate cache
→ enforce hop limit
→ check originator condition
→ schedule rebroadcast with jitter
→ decompress packet if necessary
→ deliver packet upward

---

## Identified Improvement Opportunity

The current implementation schedules rebroadcast immediately after first reception:

receive packet
→ schedule rebroadcast

This repository proposes extending this stage to:

receive packet
→ observe duplicate arrivals during jitter window
→ estimate neighborhood forwarding activity
→ suppress redundant rebroadcast when coverage is sufficient

This enables adaptive mesh-under forwarding behavior while preserving reliability.
