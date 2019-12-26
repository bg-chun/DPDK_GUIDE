# Something helpful to run DPDK inside a container

## Motivation
- As a NFV Infra developer, my mission is providing well-shaped kubernetes infra for telco applications like 5G things such as vRAN and UPF. Someguys say that now is the age of kubernetes and cloud native for 5G, but many telcos and vendors are still running their time critical packet processing applications inside VM and they just started to shift from VM to Container. For serveral months, I struggled to run DPDK inside a container based on kubernetes infra. Eventually, I found lots of things we("infra manager/developer, DPDK application developer, operator") have to know to run DPDK inside a container. So I'm ganna share it. Take a breath, here we go.

## Mandatory things to run DPDK.
- DPDK is the set of library that allows us to run packet processing in a user-space. The most significant advantage of processing packet in a userspace is enhanced performance. For this advantages, DPDK requres some dedicate things: CPU Pinning, Hugepages, Memory Lock Capability, Userspace Driver, DPDK compatible NIC and/or SR-IOV and NUMA-aware resource allocation.

## How to get these things on Kubernetes?

### CPU Pinning
- Kubernetes has CPU Manager that is internal component of Kubelet and provides exlusive cpu allocation capabilitiy.

- Configure CPU Manager
  ```
  TBD
  ```

- Request Exclusive CPU
  ```
  TBD
  ```

- How to get container's CPU Affinity?

### Hugepages support
- Kubernetes support to use pre-allocated hugepages on a node. If pre-allocated hugepages are available on a node, kubelet detects it automatically and adverties them as an allocatable node resource.

- Configure pre-allocated hugepages
  To configure pre-allocated hugepages, we have to specify default size of hugepages and quantity of pages on a boot parameters.
  For example, below will trigger to kernel to allocate 8 of 1GB hugepages on your node. If you have multi-socket machine like a two socket, kernel will allocate 4 of hugepages on each socket.
  ```
  default_hugepagesz=1G hugepagesz=1G hugepages=8
  ```
  Below is the configuration of my testing server ,which has only one socket, to test DPDK.
  ```
  GRUB_CMDLINE_LINUX="crashkernel=auto spectre_v2=retpoline rd.lvm.lv=centos_bgchun-labtop/root rhgb quiet modprobe.blacklist=nouveau default_hugepagesz=1G hugepagesz=1G hugepages=8 iommu=pt intel_iommu=on pci=assign-busses pci=realloc ixgbe.max_vfs=4"
  ```
  Then you can check that the node has 8 of pre-allocated hugepages and node adverties it.  
  
  You can easily get quantity of allocated hugepages from '/proc/meminfo'.
  ```
  cat /proc/meminfo
  ...
  HugePages_Total:       8
  HugePages_Free:        8
  ...
  ```
  
  If you have multi-socket machine and want to get quantity of allocated or free hugepages per NUMA node, try 'cat /sys/devices/system/node/node*/hugepages/hugepages-*/nr_hugepages and free_hugepages'.  
  Here is brief result from two-socket machine.
  ```
  cat /sys/devices/system/node/node0/hugepages/hugepages-1048576kB/free_hugepages 
  4
  cat /sys/devices/system/node/node0/hugepages/hugepages-1048576kB/nr_hugepages 
  4
  cat /sys/devices/system/node/node1/hugepages/hugepages-1048576kB/free_hugepages 
  4
  cat /sys/devices/system/node/node1/hugepages/hugepages-1048576kB/nr_hugepages 
  4
  ```
  
  Below result shows that kubelet on the node says that it has 8Gi of allocatable 1GB Hugepages to master node.
  ```
  kubectl describe node
  Capacity:
    cpu:                       24
    ephemeral-storage:         673705328Ki
    hugepages-1Gi:             8Gi
    intel.com/intel_sriov_pf:  2
    memory:                    130516304Ki
    pods:                      110
  Allocatable:
    cpu:                       24
    ephemeral-storage:         620886829257
    **hugepages-1Gi:             8Gi**
    intel.com/intel_sriov_pf:  2

  ```

- Request hugepages
  - Now, it is time to request hugepages for a container. To consume hugepages on Kubernetes, we have three ways to go, 1) shmget, 2) mmap with filebacking, 3) mmap with anonymous hugepages.
  In this document, I recommend to choose the second way, because that is most simple way to consume hugepages on DPDK. and you can pass the hugetlbfs mmaped file path to DPDK lib by using `--huge-dir` EAL parameter. More details for EAL parameters are available on [here](#https://doc.dpdk.org/guides/linux_gsg/linux_eal_parameters.html).
  ```
  TBD: pod spec with mmap and filebacking
  ```
- Consume hugepages with filebacking
  ```
  TBD: code
  ```
  ```
  TBD: DPDK sample, testPMD 
  ```

### How to configure NIC with/without SR-IOV

### Memory Locking Capability

### NUMA-aware resource allocation

## How to containerize DPDK App.

## Something little bit helpful to manager NFV Cluster

### Node Allocatable Feature

### CPU Reservation for non-Kubernetes stuff on Cluster

## FAQ

## Future works
