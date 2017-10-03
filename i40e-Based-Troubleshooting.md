This is a free form, almost FAQ-like, page where we can list some observed symptoms and the resolution(s)


**Symptom:** Setting strip_stag in the VF config file seemed to be ineffective.  Packets were still arriving in the guest with the VLAN ID.

**Resolution** The i40evf driver has the option to 'reconstruct' the VLAN id in the header by taking information (placed into the Rx descriptor as it was stripped by the NIC) and putting it back into the header.   This had to be specifically disabled in the guest with:  `ethtool -k <device> rxvlan off`  This feature could be referred to in documentation as _VLAN offload._