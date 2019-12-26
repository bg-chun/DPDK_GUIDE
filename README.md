# Something you should know to run DPDK inside a container

## Motivation
- As a NFV Infra developer, my mission is providing well-shaped kubernetes infra for telco applications like 5G things such as vRAN and UPF. Someguys say that now is the age of kubernetes and cloud native for 5G, but many telcos and vendors are still running their time critical packet processing applications inside VM and they just started to shift their application from VM to Container. For serveral months, I struggled to run DPDK inside a container based on kubernetes infra. Eventually, I found lots of things we("infra manager/developer, DPDK application developer, operator") should to know to run DPDK inside a container. So I'm ganna share it. Take a breath, here we go.

## Mandatory things to run DPDK.
- DPDK is the set of library that allows us to run packet processing in a user-space. The most significant advantage of processing packet in a userspace is enhanced performance. For this advantages, DPDK requres some dedicate things: CPU Pinning, Hugepages, Memory Lock Capability, Userspace Driver, DPDK compatible NIC and/or SR-IOV and NUMA-aware resource allocation.

## How to get these things on Kubernetes?

### Machine Configuration
- TBD: BIOS setting(vt-d), kernel configuration(iommu, sriov, hugepages)

### CPU Pinning
- Kubernetes has CPU Manager that is internal component of Kubelet and provides exlusive cpu allocation capabilitiy.

- Configure CPU Manager
  TBD: static policy

  ```
  TBD
  ```

- Request Exclusive CPU
  To request exclusive CPU usage, there are two things to keep in mind. First, The pod must have Guaranteed QoS class. Second, the request for exclusive CPUs must be made with the integer number.  
  The Guaranteed QoS class requires that every container in the Pod must have: 1) memory limit/request 2) CPU limit/request and they must be the same. [Here](https://kubernetes.io/docs/tasks/configure-pod-container/quality-service-pod/) is more details for QoS classes in Kubernetes.  
  Below example shows the container spec that request the integer number of exclusive CPUs and the equal amount of CPUs and memory for Guaranteed QoS Class. [Here](https://kubernetes.io/docs/tasks/administer-cluster/cpu-management-policies/#static-policy) is more examples those request exlusive CPU usage.
  ```
  spec:
  containers:
  - name: test
    image: test
    resources:
      limits:
        cpu: "2"
        memory: "1Gi"
      limits:
        cpu: "2"
        memory: "1Gi"
  ```

- How to get container's CPU Affinity?
  Sometimes we have to know which CPU cores are assigned for a container to perform thread level CPU Pinning.
  

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
  In this document, I recommend to choose the second way, because that is most simple way to consume hugepages with DPDK. More details for comsuming huegepages on Kubernetes is available on [here](https://github.com/kubernetes/enhancements/blob/master/keps/sig-node/20190129-hugepages.md#user-stories-optional).
  
    The below Pod Spec shows that requesting 2Gi of 1Gbi hugepages with filebacking.
  When user reqests hugepage with filebacking, Kubernetes creates emptyDir volume on the node then maps it to a hugetlbfs mount path of the node. It means that your container mounts hugetlbfs using Kubernetes volume system and you can create a file on it. The only left thing to do is consuming hugepages.
  
    For DPDK, you can pass the mounted path(in below example, it is `/hugepages`, see volumeMounts and Volumes section) over `--huge-dir`EAL Parameter. More details for EAL parameters are available on [here](https://doc.dpdk.org/guides/linux_gsg/linux_eal_parameters.html).
    ```
    apiVersion: v1
    kind: Pod
    metadata:
      name: testpmd-filebacking-with-mlock
    spec:
      containers:
      - name: testpmd
        image: bgchun/testpmd
        command: [ "/bin/bash", "-c", "--" ]
        args: [ "while true; do sleep 300000; done;" ]
        securityContext:
         capabilities:
           add: ["IPC_LOCK"]
        volumeMounts:
        - mountPath: /hugepages
          name: hugepage
        resources:
          requests:
            memory: "1Gi"
            hugepages-1Gi: "2Gi"
            intel.com/intel_sriov_pf: '1'
          limits:
            memory: "1Gi"
            hugepages-1Gi: "2Gi"
            intel.com/intel_sriov_pf: '1'
      volumes:
      - name: hugepage
        emptyDir:
          medium: HugePages
    ```
    For Non-DPDK application, as i mentioned above, there is three ways to consume hugepages.
    Below example shows comsuming hugepages using filebacking, It is very simple: open & mmap.
    Below is the part of mmap sample on linux kernel. See [here](https://github.sec.samsung.net/RS7-EdgeComputing/hugepage-samples/blob/master/poc/mmap_filebacking.cc) to get the smaple.
    There are two more smaples those consume hugepages over shmget(see [here](https://github.com/torvalds/linux/blob/master/tools/testing/selftests/vm/hugepage-shm.c)) and mmap with anonymous mapping(see [here](https://github.com/torvalds/linux/blob/master/tools/testing/selftests/vm/map_hugetlb.c)).
    ```
    int fd = open(/*FILE under hugetlbfs*/, O_CREAT | O_RDWR, 0755);
  
    // On success, mmap() returns a pointer to the mapped area.
    // On error, the value MAP_FAILED is returned, and errno is set to indicate the cause of the error.
    // http://man7.org/linux/man-pages/man2/mmap.2.html
    void *addr = mmap(NULL, (size), PROT_READ | PROT_WRITE, MAP_SHARED, fd, 0);
    ```

### How to configure NIC with/without SR-IOV
- To run DPDK, you have to prepare DPDK compatible NIC on your server machine. The list of compatibe devices can be found [here](https://core.dpdk.org/supported/).

### Memory Locking Capability
- To run DPDK, we need IPC_LOCK capability.
  TBD: you need IPC_LOCK or priviliged power


### NUMA-aware resource allocation

## How to containerize DPDK App.
TBD: here is the best example: https://github.com/pliurh/Kubecon2019-DEMO/blob/master/Dockerfile

## Something little bit helpful to manager NFV Cluster

### Node Allocatable Feature

### CPU Reservation for non-Kubernetes stuff on Cluster

## FAQ

## Future works
