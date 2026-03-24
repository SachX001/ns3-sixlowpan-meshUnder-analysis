# ns3-sixlowpan-meshUnder-analysis

## Adaptive Mesh-Under Flooding Improvements for ns-3 6LoWPAN
 This repository documents analysis and proposed improvements to the mesh-under flooding implementation in the ns-3 6LoWPAN module.
 The goal is to improve broadcast efficiency by introducing duplicate-rate-aware rebroadcast suppression during the jitter window inside:
 $${\\color{red}SixLowPanNetDevice::ReceiveFromDevice()}$$   

## Motivation
 The current mesh-under forwarding implementation schedules rebroadcast after the first packet reception using randomized jitter but does not consider neighborhood forwarding activity.
 This can result in redundant transmissions in dense mesh networks.

## Proposed Improvement
 Introduce adaptive rebroadcast suppression based on duplicate arrival rate observed during jitter interval.
 Nodes suppress forwarding when sufficient neighborhood coverage is detected.

## Repository Structure

 ###docs/
  → explanation of existing implementation

 ###pseudocode/
  → comparison of current vs improved forwarding logic

 ###topology-diagrams/
  → broadcast scenarios

 ###experiments/
  → evaluation strategy
