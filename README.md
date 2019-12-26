# Something helpful to run DPDK inside a container

## Motivation
- As a NFV Infra developer, my mission is providing well-shaped kubernetes infra for telco applications like 5G things such as vRAN and UPF. Someguys say that now is the age of kubernetes and cloud native for 5G, but many telcos and vendors are still running their time critical packet processing applications inside VM and they just started to shift from VM to Container. For serveral months, I struggled to run DPDK inside a container based on kubernetes infra. Eventually, I found lots of things we("infra manager/developer, DPDK application developer, operator") have to know to run DPDK inside a container. So I'm ganna share it. Take a breath, here we go.

## Mandatory things to run DPDK.
- DPDK is the set of library that allows us to run packet processing in a user-space. The most significant advantage of processing packet in a userspace is enhanced performance. For this advantages, DPDK requres some dedicate things: CPU Pinning, Hugepages, Memory Lock Capability, Userspace Driver, DPDK compatible NIC and/or SR-IOV and NUMA-aware resource allocation.

## How to get these things on Kubernetes?

### CPU Pinning
- Kubernetes has CPU Manager that is internal component of Kubelet and provides exlusive cpu allocation capabilitiy.

- Configure CPU Manager

- Request Exclusive CPU

- How to get container's CPU Affinity?

### Hugepages support
- Kubernetes support to use pre-allocated hugepages on a node, 

### How to configure NIC with/without SR-IOV

### Memory Locking Capability

### NUMA-aware resource allocation

## Something little bit helpful to manager NFV Cluster

### Node Allocatable Feature

### CPU Reservation for non-Kubernetes stuff on Cluster

## FAQ

## Future works
