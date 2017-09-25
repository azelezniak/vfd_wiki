     
 
__________________________________________________________________
CAUTION: 
This document is generated output from {X}fm source. Any 
modifications made to this file will certainly be overlaid by 
subsequent revisions. The proper way of maintaining the document 
is to modify the source file(s) in the repo, and generating the 
desired .pdf or .md versions of the document using {X}fm. 
__________________________________________________________________
**Virtual Function Daemon -- VFd** 

**Debugging Tricks** 
 
 
# Overview 
VFd is a daemon which manages the configuration of virtual and 
physical functions (VFs and PFs) which are exposed by an SR-IOV[1] 
capable network interface card (NIC). Configuration and management 
is accomplished via DPDK which requires a DPDK enabled driver be 
attached to both the NIC PF and VFs which are to be made 
accessible to the guests through SR-IOV. Once a DPDK driver is 
bound to the PF and VFs, the usual debugging tools (e.g. tcpdump) 
become unavailable. This document is a set of _tricks_ that might 
make sussing out the cause of networking issues when a guest is 
attempting to use an SR-IOV provided device. 
 
 
# Using Mirrors 
As of August 2017 mirroring traffic in and out of a VF is 
supported provided that the underlying NICs support it (Niantic is 
confirmed). When it appears that no traffic is arriving, or being 
transmitted, to/from a VF, mirroring the traffic to another VF, 
and running a tool like <code>tcpdump</code> in the other VF can 
be useful. 
 
 
## Configuring The PF 
To support mirroring, the _enable_loopback_ setting in the main 
VFd configuration must be set for the PF that owns the VFs that 
the guests will use. If this is not set, the outbound traffic from 
the troubled guest will not be visible on the debugging guest. 
 
 
## Configuring The Debugging Guest 
The debugging guest is where the mirrored traffic will be sent. It 
is configured much like any other guest though the setting for 
unknown unicast (allow_un_ucast" **must** be set to false. The 
following is a sample configuration for a VF that was used as the 
mirror destination. 
 
    {
    "comments":      "",
 
    "name":             "daniels_0_3",
    "pciid":            "0000:08:00.0",
    "vfid":             3,
    "strip_stag":       true,
    "allow_bcast":      true,
    "allow_mcast":      true,
    "allow_un_ucast":   false,
 
    "vlan_anti_spoof":  false,
    "mac_anti_spoof":   false,
    "vlans":            [ 101 ],
    "macs":             [ "fa:ce:de:02:00:03" ],
 
    "queues": [
        { "priority": 0, "share": "90%" },
        { "priority": 1, "share": "90%" },
        { "priority": 2, "share": "90%" },
        { "priority": 3, "share": "90%" },
    ]
    }
Figure 1: Sample configuration for a target VF. 
 
 
 
## Configuring The Problem Guest 
The configuration of the problem guest should be as established by 
the virtualisation manager, however one additional configuration 
parameter will need to be added in order to start the mirroring. 
The mirroring definition, illustrated below, can be added as the 
last parameter to the existing configuration file. Once it is 
added, the configuration will need to be reloaded; this process is 
discussed next. 
 
    "mirror": { "target": 3, "direction": "all" }
 
Figure 2: The mirroring configuration json bits added to the 
troubled guest's configuration file. 
 
The _target_ is the VF number of the debugging guest. This may not 
be known until the guest is started (depending on whether the 
guest is being started manually or via some virtualisation 
driver). The direction may be one of four strings: 
 
all  Mirror all traffic to the target. 
in   Mirror only traffic received and passed to the guest. 
out  Mirror only traffic which is transmitted from the guest. 
off  Turn mirroring off. 
 
 
 
 
## Changing Mirroring State 
To change the mirror state for a running guest involves a set of 
simple steps: 
 
1         Make a back up of the current configuration of the VF 
         configuration (/var/lib/vfd/config). 
          
2         Add the mirroring setting to the backup configuration 
         setting in, our, all, or off as desired. 
          
3         Delete the current configuration from VFd (iplex delete 
         <id>). The configuration file will be removed. 
          
4         Copy the backup file to the config directory giving it 
         the same name as before. 
          
5         Add the newly created configuration to VFd (iplex add 
         <id>). 
          
6         Verify in the VFd log file that mirroring state was 
         changed for the desired VF. 
 
 
 
 
## Detecting Configuration Issues 
The mirroring technique works only if the traffic arriving will 
actually be passed into the problematic guest, or the traffic 
being sent is actually accepted by the NIC for transmission. In 
cases where traffic is not flowing because the VLAN and/or MAC 
address is/are not correctly configured the traffic will also not 
be mirrored as the traffic would not normally be accepted by the 
NIC. These are classic cases where the spoof counter is increasing 
with the Rx/Tx count at the PF level. 
 
Mirroring will help with misconfiguration when stripping is 
involved. If the _strip_stag_ setting is set to false when the 
guest cannot handle packets with the VLAN ID still in place, or 
the outer VLAN ID in place, the packet will be mirrored to the 
target guest and should provide a clue about the configuration 
problem. 
 
 
# Notes 
 
 
 
__________________________________________________________________
 
[1] A basic primer on SR-IOV can be found at 
xxxx://www.intel.com/content/dam/doc/application-note/pci-sig-sr-iov-primer-sr-iov-technology-paper.pdf 
 
 
*** 
__________________________________________________________________
 
**Source:** vfd/doc/debug.xfm 
 
**Original:** 25 September 2017 
 
**Revised:** 25 September 2017 
 
**Formatter:** tfm V2.2/0a266 
 
