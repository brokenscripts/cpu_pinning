# CPU Pinning  
***This guide is related to VFIO and/or CPU pinning for NICs.***  


The default behavior for KVM guests is to run operations coming from the guest as a number of threads representing virtual processors. Those threads are managed by the Linux scheduler like any other thread and are dispatched to any available CPU cores based on niceness and priority queues. As such, the local CPU cache benefits (L1/L2) are lost each time the host scheduler reschedules the virtual CPU thread on a different physical CPU. This can noticeably harm performance on the guest. CPU pinning aims to resolve this by limiting which physical CPUs the virtual CPUs are allowed to run on. The ideal setup is a one to one mapping such that the virtual CPU cores match physical CPU cores while taking hyperthreading/SMT into account.<sup>[1]</sup>  
> **Note**: For certain users enabling CPU pinning may introduce stuttering and short hangs, especially with the MuQSS scheduler (present in linux-ck and linux-zen kernels). You might want to try disabling pinning first if you experience similar issues, which effectively trades maximum performance for responsiveness at all times.<sup>[1]</sup>  

It's very important that when we passthrough a core, we include its sibling.  A matching core id (i.e. "CORE" column) means that the associated threads (i.e. "CPU" column) run on the same physical core.<sup>[2]</sup>  

CPU pinning is a process where-in a vCPU is tied to a physical CPU core or thread. On systems with multiple NUMA nodes (which is often the case with multi-socket systems or multi-die processors) this prevents requests to/from memory (RAM) from having to cross Intels QPI link or AMDs Infinity Fabric to answer said request. The result is a noticeable performance boost under certain applications.<sup>[4]</sup>  
> **NOTE**: This is not the case for all systems/processors, in many instances the system will utilize UMA where there is only one CPU and one bank of memory. Do your research on your hardware. Now pinning the vCPUs on UMA systems won't hurt anything and if there's still anything to be had it may still be worth doing.<sup>[4]</sup>  

For example on a Ryzen Threadripper 1950X. This is a two die 16 core processor with two NUMA Nodes. Although the hardware knows this the BIOS manufacturers will frequently set a feature known as Memory Interleave to Auto. This creates an layer of abstraction which makes the NUMA nodes operate like a UMA Node. It's important to change this from Auto to Channel.<sup>[4]</sup> 

## CPU topology

Most modern CPUs support hardware multitasking, also known as hyper-threading on Intel CPUs or SMT on AMD CPUs. Hyper-threading/SMT is simply a very efficient way of running two threads on one CPU core at any given time. You will want to take into consideration that the CPU pinning you choose will greatly depend on what you do with your host while your VM or service(s) are running.<sup>[1]</sup>  

### Tools

#### lscpu (`util-linux` package)  
To find the topology for your CPU run **`lscpu -e`**  
> **Note**: Pay special attention to the 4th column "**CORE**" as this shows the association of the Physical/Logical CPU cores  


##### AMD example<sup>[1]</sup>
`lscpu -e` on a 6c/12t Ryzen 5 1600: 
```bash
CPU NODE SOCKET CORE L1d:L1i:L2:L3 ONLINE MAXMHZ    MINMHZ
0   0    0      0    0:0:0:0       yes    3800.0000 1550.0000       # Core 0 sequential, on 0 & 1
1   0    0      0    0:0:0:0       yes    3800.0000 1550.0000       # Core 0 sequential, on 0 & 1
2   0    0      1    1:1:1:0       yes    3800.0000 1550.0000
3   0    0      1    1:1:1:0       yes    3800.0000 1550.0000
4   0    0      2    2:2:2:0       yes    3800.0000 1550.0000
5   0    0      2    2:2:2:0       yes    3800.0000 1550.0000
6   0    0      3    3:3:3:1       yes    3800.0000 1550.0000
7   0    0      3    3:3:3:1       yes    3800.0000 1550.0000
8   0    0      4    4:4:4:1       yes    3800.0000 1550.0000
9   0    0      4    4:4:4:1       yes    3800.0000 1550.0000
10  0    0      5    5:5:5:1       yes    3800.0000 1550.0000
11  0    0      5    5:5:5:1       yes    3800.0000 1550.0000
```

> **Note**: Ryzen 3000 ComboPi AGESA changes topology to match Intel example, even on prior generation CPUs. Above valid only on older AGESA. 

##### Intel example<sup>[1]</sup>
`lscpu -e` on a 6c/12t Intel 8700k: 
```bash
CPU NODE SOCKET CORE L1d:L1i:L2:L3 ONLINE MAXMHZ    MINMHZ
0   0    0      0    0:0:0:0       yes    4600.0000 800.0000        # Core 0 non-sequential, on 0 & 6
1   0    0      1    1:1:1:0       yes    4600.0000 800.0000
2   0    0      2    2:2:2:0       yes    4600.0000 800.0000
3   0    0      3    3:3:3:0       yes    4600.0000 800.0000
4   0    0      4    4:4:4:0       yes    4600.0000 800.0000
5   0    0      5    5:5:5:0       yes    4600.0000 800.0000
6   0    0      0    0:0:0:0       yes    4600.0000 800.0000        # Core 0 non-sequential, on 0 & 6
7   0    0      1    1:1:1:0       yes    4600.0000 800.0000
8   0    0      2    2:2:2:0       yes    4600.0000 800.0000
9   0    0      3    3:3:3:0       yes    4600.0000 800.0000
10  0    0      4    4:4:4:0       yes    4600.0000 800.0000
11  0    0      5    5:5:5:0       yes    4600.0000 800.0000
```

As we see above, with **AMD Core 0** is sequential with **CPU 0 & 1**, whereas the **Intel Core 0** is on **CPU 0 & 6**. 

If you do not need all cores for the guest, it would then be preferable to leave at the very least one core for the host. Choosing which cores one to use for the host or guest should be based on the specific hardware characteristics of your CPU, however **Core 0** is a good choice for the host in most cases. If any cores are reserved for the host, it is recommended to pin the emulator and iothreads, if used, to the host cores rather than the VCPUs. This may improve performance and reduce latency for the guest since those threads will not pollute the cache or contend for scheduling with the guest VCPU threads. If all cores are passed to the guest, there is no need or benefit to pinning the emulator or iothreads.<sup>[1]</sup> 

#### lstopo (`hwloc` package)  
The package `hwloc` can visually show you the topology of your CPU.<sup>[2]</sup>  
To find the topology for your CPU run **`lstopo`** for a graphical window or **`lstopo --of txt`** for a CLI picture<sup>[2]</sup>  

![](images/2021-04-16-14-24-41.png)  

The format above can be a bit confusing due to the default display mode of the indexes.  Toggle the display mode using **`i`** until the legend (at the bottom) shows "**Indexes: Physical**". The layout should become more clear. In my case it becomes this:<sup>[2]</sup>

![](images/2021-04-16-14-25-47.png)  

To explain a little bit, I have 6 physical cores (Core P#0 to P#6) and 12 virtual cores (PU P#0 to PU P#11). The 6 physical cores are mainly divided into two sets of 3 cores: Core P#0 to P#2; and Core P#4 to P#6. Each group has its own L3 cache. However, the most important thing to pay attention here is how virtual cores are mapped to the physical core. The virtual cores (notated PU P#...) come in pairs of two i.e. siblings:<sup>[2]</sup>

*  PU P#0 and PU P#6 are siblings in Core P#0
*  PU P#1 and PU P#7 are siblings in Core P#1
*  PU P#2 and PU P#8 are siblings in Core P#3


## PCI layout  
The command `lspci -vnn` can be used to display which NUMA Node a PCI device (GPU, NIC, etc) is connected to. Just search for the Device ID or Device Address. It will list the NUMA Node.  
```
# lspci -vnn
...
00:1f.6 Ethernet controller [0200]: Intel Corporation Ethernet Connection (2) I219-V [8086:15b8]
	Subsystem: Micro-Star International Co., Ltd. [MSI] Device [1462:7a66]
	Flags: bus master, fast devsel, latency 0, IRQ 137, IOMMU group 11
	Memory at df300000 (32-bit, non-prefetchable) [size=128K]
	Capabilities: <access denied>
	Kernel driver in use: e1000e
	Kernel modules: e1000e

...
```
### Method 1
In a multi-NUMA layout, its easier to identify which NUMA the NIC is tied to:  
Using a `lstopo` output from [Open MPI,Xeon](https://www.open-mpi.org/projects/hwloc/lstopo/images/2XeonE5v3.v1.11.png):  
> 2x Xeon Haswell-EP E5-2683v3 (from 2014, with hwloc v1.11).
> Processors are configured in Cluster-on-Die mode which shows 2 NUMA nodes per package
![](images/2021-04-16-14-56-45.png)  

This picture shows that NUMA 0, Core P0 - P6 are where the NIC resides.  


### Method 2
Manually go to `/sys/class/net/{DEVICE}/device/`.  Replace {DEVICE} with your NIC name from `ip link` command (example: eth0, or enp0s31f6).  
Or `/sys/bus/pci/devices/0000:{DEVICE_ID}/`.  Replace {DEVICE_ID} with the DEVICE_ID identified via `lspci -vnn` (example: 00:1f.6).  

A listing of files at this path will be:  
```
ari_enabled               dma_mask_bits    irq            net          reset             uevent
broken_parity_status      driver           link           numa_node    resource          vendor
class                     driver_override  local_cpulist  power        resource0         wakeup
config                    enable           local_cpus     power_state  revision
consistent_dma_mask_bits  firmware_node    modalias       ptp          subsystem
d3cold_allowed            iommu            msi_bus        remove       subsystem_device
device                    iommu_group      msi_irqs       rescan       subsystem_vendor
```
`numa_node` *might* show the numa node connected.  
`local_cpulist` and `local_cpus` *might* give more information as well.  


## NIC NUMA domain<sup>[5]</sup>  
Accessing NIC in the same NUMA domain is faster than across NUMA domain<sup>[6]</sup>.  Example:
![](images/2021-04-16-15-06-52.png)  
In summary, always use cores with NIC within the same NUMA node if possible to gain best performance when pinning CPUs.  


[1]: https://wiki.archlinux.org/index.php/PCI_passthrough_via_OVMF#CPU_pinning  
[2]: https://gitlab.com/Karuri/vfio/#tool-2-hwloc-for-a-more-in-depth-understanding  
[3]: https://github.com/bryansteiner/gpu-passthrough-tutorial#----cpu-pinning
[4]: https://linustechtips.com/topic/1156185-vfio-gpu-pass-though-w-looking-glass-kvm-on-ubuntu-1904/
[5]: http://docplayer.net/5271505-Network-function-virtualization-virtualized-bras-with-linux-and-intel-architecture.html
[6]: https://stackoverflow.com/questions/28307151/is-cpu-access-asymmetric-to-network-card