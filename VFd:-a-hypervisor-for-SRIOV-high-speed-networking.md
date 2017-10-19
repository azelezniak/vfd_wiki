# News

* Sept 2017: The VFd talk was accepted and presented at the DPDK Summit in Dublin: [Slides](https://dpdksummit.com/Archive/pdf/2017Userspace/DPDK-Userspace2017-Day2-3-VFd.pdf)

# About VFd

VFd (SR-IOV Virtual Function driver) is a daemon which manages the configuration of virtual and physical functions (VFs and PFs) which are exposed by an SR-IOV capable network interface card (NIC) using the DPDK PMD interface. Configuration and management is accomplished via DPDK and allows a very lightweight and simple overlay to be constructed for cloud environments. Currently, the following attributes can be controlled:

* VLAN and MAC filtering (inbound)
* Stripping of VLAN ID (single) from inbound traffic
* Addition of VLAN ID (single) to outbound traffic
* Enabling/disabling broadcast, multicast, unicast

In the future, the following additional capabilities are being developed for configuration through VFd

* Network QoS: bandwidth allocation, prioritization, and rate limiting
* Improved analytics interface to report basic packet level statistics on PF and VFs

Recently completed enhancements:

* Packet mirroring from one VF to another

Currently, VFd only works with Intel 82599 NICs (Niantic). However, the software is intended to be NIC agnostic going forward, with support for additional NICs coming in the future. 

# Contacts

The github site will be the location where most VFd design and user documentation will be archived. In addition, VFd development can be followed at the following places.

VFd Users mailing list: to subscribe, send email to vfd-users+subscribe@googlegroups.com with the word subscribe in the subject line or <a href="mailto:vfd-users+subscribe@googlegroups.com?subject=subscribe">Click Here</a>.

Mailing list archives can be found here: https://groups.google.com/forum/#!forum/vfd-users