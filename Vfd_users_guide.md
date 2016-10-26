 
__________________________________________________________________
CAUTION: 
This document is generated output from {X}fm source. Any 
modifications made to this file will certainly be overlaid by 
subsequent revisions. The proper way of maintaining the document 
is to modify the source file(s) in the repo, and generating the 
desired .pdf or .md versions of the document using {X}fm. 
__________________________________________________________________

**Virtual Function Daemon -- VFd** 
 
**User's Guide** 
 
 
# Overview 
VFd is a daemon which manages the configuration of virtual and 
physical functions (VFs and PFs) which are exposed by an SR-IOV[1] 
capable network interface card (NIC). Configuration and management 
is accomplished via DPDK and allows for the following attributes 
to be controlled: 
 
* VLAN and MAC filtering (inbound) 
  
* Stripping of VLAN ID (single) from inbound traffic 
  
* Addition of VLAN ID (single) to outbound traffic 
  
* Enabling/disabling broadcast, multicast, unicast 
 
 
 
The configuration information for each VF under the control of VFd 
is supplied via a set of JSON encoded parameters, one parameter 
set per file, located in the /var/lib/vfd directory. An overall 
configuration file for the daemon, also a set of JSON encoded 
parameters, is managed in the /etc/vfd directory. The command line 
tool iplex is used to communicate with VFd and provides the 
mechanism to add or remove VF configurations. 
 
Under normal circumstances VFd and any related software tools are 
installed on the system via some distribution mechanism (chef, 
puppet, or the like), which is also responsible for the creation 
of the configuration file in /etc and ensures that the VFs have 
been created. Nova, or some similar VM management environment is 
generally responsible for creating the individual VF 
configurations, placing them into the /var directory, and using 
iplex to add the configuration information as a VM is created. 
 
At times it may be necessary to install and make use of VFd 
without the usual distribution and virtual environment management 
components. This guide provides a way to hack together such a 
system and describes what is required to 
 
* Create the Virtual Functions associated with a Physical Function 
  
* Bind and unbind a VF or PF to/from a driver 
  
* Install the VFd components (.deb based) 
  
* Create the VFd configuration file 
  
* Create a VF configuration file and cause it to be used to 
 configure a VF 
  
* Start and stop the daemon 
 
 
 
 
 
# Physical and Virtual Functions 
Before VFd can be started, the PF which will provide VFs must be 
identified and the VFs must be created. The lspci command, 
generally using grep to limit the output to ethernet devices, can 
be used to identify the NICs on the system, and from that list the 
PFs which need to be set up with VFs can be determined and the VFs 
added (see the _Limitations and Requirements_ section later in 
this document for information regarding the number of VFs which 
must be created for each PF). 
 
 
## Adding VFs to a PF 
Assuming the PF is 0000 the following can be used to add VFs under 
it. 
 
1 Cd to the device directory. (e.g. /sys/devices/pci0000). 
  
2 Disable all current VFs (echo 0 >sriov_numfs) 
  
3 Enable the number of VFs desired (echo 32 >sriov_numfs) 
  
4 Verify that the VFs were added (ls -al in the directory and 
 expect to find symbolic links like virtfn3->../0000) 
 
 
 
 
The  above  example shows the creation of 32 VFs; up to 32 VFs may 
be added to a single PF. As a final step, the ixgbe driver  should 
be  unbound  from the PF and VFs, and the vfio-pci driver bound to 
each. The dpdk_nic_bind utility (installed with VFd) can  be  used 
to do this as illustrated in figure 1. 
 
     
     
     
           dpdk_nic_bind -u 0000:01:00.1
           dpdk_nic_bind -b vfio-pci 0000:01:10.7
 
 
 
Figure  2:  Commands  used to unbind and bind drivers from PFs and 
VFs. 
 
Note that only the PCI address is needed for these commands. 
 
 
## PF/VF Verification 
The dpdk_nic_bind utility can also be used to verify the state  of 
the  various  network interfaces on the system. Executing with the 
--status option will cause it to list all network  interfaces.  If 
the  VFs  were  added, and bound to the correct driver, the output 
from the command should  list  them  all  under  a  heading  which 
indicates  that  they are bound to a DPDK capable driver. Figure 2 
illustrates the status output from this command and shows that VFs 
have  been  added  to  the  PF,  but are not yet bound to vfio-pci 
driver (they do not show under the DPDK header). 
     
     
     
         
         Network devices using DPDK-compatible driver
         ============================================
         <none>
         
         Network devices using kernel driver
         ===================================
         0000:02:00.0 'NetXtreme BCM5720 Gigabit Ethernet PCIe' if=eth0 drv=tg3 unused=vfio-pci *Active*
         0000:02:00.1 'NetXtreme BCM5720 Gigabit Ethernet PCIe' if=eth1( drv=tg3 unused=vfio-pci 
         
         Other network devices
         =====================
         0000:08:00.0 'Ethernet 10G 2P X520 Adapter' unused=vfio-pci
         0000:08:00.1 'Ethernet 10G 2P X520 Adapter' unused=vfio-pci
         0000:08:10.0 '82599 Ethernet Controller Virtual Function' unused=vfio-pci
         0000:08:10.1 '82599 Ethernet Controller Virtual Function' unused=vfio-pci
         0000:08:10.2 '82599 Ethernet Controller Virtual Function' unused=vfio-pci
         0000:08:10.3 '82599 Ethernet Controller Virtual Function' unused=vfio-pci
         0000:08:10.4 '82599 Ethernet Controller Virtual Function' unused=vfio-pci
         0000:08:10.5 '82599 Ethernet Controller Virtual Function' unused=vfio-pci
 
 
 
Figure 3: Partial output generated by  the  dpdk_nic_bind  utility 
with  the  --status  option. VFs had been created but not bound to 
the driver. 
 
 
 
## IOMMU Groups 
If a NIC has  two  physical  interfaces,  these  will  be  grouped 
together  from  an IOMMU perspective, and in order for VFd to have 
access to the NICs, all of the devices in a group must be bound to 
a DPDK capable driver. If all devices belonging to a group are not 
bound to the DPDK capable driver, an error will  be  generated  as 
VFd attempts to initialise, and the DPDK library will abort. 
 
Given  a  known  PF, it is simple to determine which other devices 
also belong to the same group. Figure 3  shows  a  simple  set  of 
commands[2]  which can be used to find the location of a known PCI 
device and then all other devices in the same  group.  The  output 
from  the  command shows the group number as the last directory in 
the path which is the group number. A subsequent list command  can 
then be used to list all devices in that group. 
 
     
     
     
           find sys/kernel/iommu_groups -name 0000:01:00.1 | read p junk
           ls ${p%/*}
 
 
 
Figure 4: Finding iommu group members. 
 
 
 
# Installation And Configuration 
VFd  is  distributed in a .deb package and can be installed with a 
simple   dpkg  command.  The  installation  process  creates  the  
configuration  directory  in  /etc  (adding a sample configuration 
file), creates the  necessary  directories  in  the  /var/lib  and 
/var/log  directories,  and  places  the following programmes into 
/usr/bin 
 
 
**vfd:** The daemon itself (64 bit Linux binary) 
 
**iplex:** The command line tool  used  to  communicate  with  VFd 
(python 2.x) 
 
**vfd_pre_start:**  The  script  which manages the startup process 
(python) 
 
**dpdk_nic_bind:** An opensource programme which  facilitates  the 
binding  and  unbinding  of  drivers  to/from virtual and physical 
functions (python) 
 
 
 
 
The installation process also  installs  the  necessary  _upstart_ 
mechanism  allowing  the  familiar _service_ command to be used to 
start and stop VFd. 
 
 
## Configuration 
The configuration file, vfd.conf,  must  be  created  in  /etc/vfd 
before VFd can be started. The file contains a set of JSON encoded 
variables which are illustrated in figure 4. 
     
     
     
     
          {
            "comments": [
              " test config file for vfd",
              " manually generated for testing 2016/03/10",
              " revised for agave207 2016/04/06"
            ],
         
            "fifo": "/var/lib/vfd/request",
            "log_dir": "/var/log/vfd",
            "log_keep": 60,
            "init_log_level": 2,
            "log_level": 2,
            "config_dir": "/var/lib/vfd/config",
            "cpu_mask": "0x04",
            "dpdk_log_level": 2,
            "dpdk_init_log_level": 8,
            "default_mtu": 1500,
         
            "pciids": [ 
               { "id": "0000:01:00.0", "mtu": 9000, "enable_loopback": false }, 
               { "id": "0000:01:00.1", "mtu": 9000, "enable_loopback": false } 
            ]
           }
 
 
 
Figure 5: A sample vfd.cfg configuration file. 
 
 
Items worth noting in the config file are: 
 
 
**fifo:**   This  is  a  named  pipe  which  iplex  uses  to  
communicate requests to VFd 
 
**log  levels:**  The verbosity of running chatter emitted by 
VFd can be controlled by these  settings.  Four  options  are 
provided  which  control the chattiness during initialisation 
(usually more information  is  desired)  and  a  level  which 
affects  the  drivel  after  initialisation  is complete. Log 
levels are supplied for both VFd  proper  and  for  the  DPDK 
library functions as it is useful to control them separately. 
 
**config_dir:** This is the directory where the individual VF 
configuration files are expected to be placed. The default is 
shown in the example. 
 
**cpu_mask:**  This is a single bit which indicates which CPU 
the DPDK library will associate with. VFd limits  this  value 
to  a  single  bit,  and odd results may occur if a multi-bit 
integer is given. 
 
**pciids:** Explained in the following section 
 
 
 
 
 
 
### Pciid Array 
The array  of  pciids  supplied  in  the  configuration  file 
defines the PFs which VFd will attempt to manage. In addition 
to the PCI address, the following value should be supplied: 
 
 
**mtu:** The mtu value for  the  device.  If  omitted  the  _ 
default_mtu_ value described earlier is used. 
 
**enable_loopback:**  When  set to true allows the NIC to act 
as a bridge and permits traffic from one  guest  to  another, 
hosted  on  the  same  PF, to be looped directly back without 
going out on the physical wire. The default value is false. 
 
 
 
 
 
 
### Comments And Other Elements 
Any field which VFd doesn't expect is ignored, and  thus  the 
comments   array   allows   the   creator  to  document  the  
configuration without  interfering  with  VFd  Order  of  the 
fields  is  unimportant,  however  the  field  names are case 
sensitive. 
 
 
# Running VFd 
The daemon can be started  manually,  or  using  the  upstart 
start and stop commands as would be done for any other system 
service. When running manually the user must be  _root,_  and 
various  command  line options may be provided allowing for a 
diverse testing environment. The VFd command line syntax is: 
 
     
     
     
           vfd [-f] [-n] [-p config-parm-file] 
 
 
 
 
Where: 
 
 
**-f:** Causes VFd to run without detaching from the tty 
(foreground) 
 
**-n:** Enables _no-execution_ mode which prevents VFd from 
actually doing anything through DPDK 
 
**-p:** Provides the filename of the configuration file 
(default is /etc/vfd/vfd.cfg) 
 
 
 
 
 
 
[ IMAGE ] 
 
 
The illustration in figure 5 shows the  relationship  of  the 
daemon  with  the iplex utility, configuration files, and the 
assumed interaction with nova. 
 
 
## VF Configuration 
In a normal  operation  environment  nova,  or  some  similar 
virtualisation manager, would create a configuration file for 
each VM and place that file into the configuration  directory 
named  in  the  main  VFd  config  file.  When  hacking  the  
environment, the hacker is  responsible  for  creating  these 
files,  adding  them  to  the config directory, and using the 
iplex command line tool to cause VFd to parse  and  configure 
the NIC(s) based on the file contents. The following sections 
describe this process and  provide  the  details  of  the  VF 
configuration files. 
 
 
### Internal configuration management 
VFd manages only the VFs which have been added to its view of 
the world; if a configuration file  is  not  provided  for  a 
PF/VF  combination,  VFd  makes no attempt to manage that VF. 
This allows for VFs  to  be  managed  and/or  used  by  other 
elements on the system (contrail vRouter?). VFd will continue 
to manage (configure) a VF until it is removed from view. 
 
 
### The VF configuration file 
The VF configuration file is a simple  set  of  JSON  encoded 
parameters  similar to the main VFd configuration in the /etc 
directory. Figure 5 provides  a  sample  configuration  which 
contains the following information: 
 
     
     
     
         {
           "comments": [
             " configure VF 1 on the pciid given",
             " comments are ignored"
           ],
          
           "name":             "VM_daniels1/uuid-dummy-daniels-1",
           "pciid":            "9999:00:00.0",
           "vfid":             1,
           "strip_stag":       true,
           "insert_stag":      true,
           "allow_bcast":      true,
           "allow_mcast":      true,
           "allow_un_ucast":   true,
           "start_cb":         "/var/lib/vmman/hot_plug daniels-1",
           "stop_cb":          "/var/lib/vmman/unplug daniels-1",
           "vlans":            [ 10, 11, 12, 33 ],
           "macs":             [ "aa:bb:cc:dd:ee:f0", 
                                 "11:22:33:44:55:66", 
                                 "ff:ff:ff:ff:ff:ff" ]
         }
 
 
 
Figure  6:  Sample  VF configuration file normally created by 
nova. 
 
 
**name:**   Is   any   string   provided  by  the  user  for  
identification of the configuration file. 
 
**pciid:**  is  the ID of the **PF** that the VF is allocated 
from. 
 
**vfid:** Is the ID  number  (0-31)  of  the  VF  on  the  PF 
supplied by the pciid. 
 
**vlans:** Is an array of one or more VLAN IDs which VFd will 
configure on the VF. 
 
**macs:** Is an array of one or more MAC addresses which  VFd 
will  configure  on the VF. It is generally **not** necessary 
to configure any MAC addresses. 
 
**start_cb:** Is a command (script) which VFd will execute as 
the  last  step  in the VFd start up process. (See section on 
callback command hooks below.) 
 
**stop_cb:** Is a command (script) which VFd will execute  as 
the first step in the VFd shutdown process. 
 
 
 
 
 
 
### Adding a configuration 
The  VF  configuration  files  are  placed into the directory 
named in  the  main  VFd  configuration  file,  generally  in 
/lib/vfd  and  are  named with any prefix and a .json suffix. 
Once a configuration file is placed into the  directory,  the 
iplex  command  line tool is used to add the configuration to 
VFd. The overall syntax for the iplex  command  is  given  in 
figure 6, with an example of an add command, to add VM1.json, 
is illustrated in figure 7. 
 
     
     
     
          Usage:
          iplex (add | delete) <port-id> [--loglevel=<value>] [--debug]
          iplex show (all | <port-id>) [--loglevel=<value>] [--debug]
          iplex verbose [--loglevel=<value>]
          iplex ping
          iplex -h | --help
          iplex --version
         
          Options:
              -h, --help      show this help message and exit
              --version       show version and exit
              --debug         show debugging output
              --loglevel=<value>  Default logvalue [default: 0]
 
 
 
Figure 7: The usage output from the iplex command. 
 
     
     
     
           iplex add VM1
 
 
 
Figure 8: Sample iplex command to add a file  named  VM1.json 
to VFd's direct control. 
 
If  the  loglevel  value  is  given  on  the command line the 
logging that VFd does during the processing  of  the  command 
will  be  increased  to  this level. If the loglevel value is 
lower than the current setting in  VFd,  then  the  value  is 
ignored. 
 
 
### Deleting a configuration 
A  configuration  added will remain in use by VFd until it is 
deleted. The iplex delete command is used  to  cause  VFd  to 
eliminate  the configuration from its view and to delete JSON 
file from the configuration directory. When using  VFd  in  a 
hacked  environment,  care must be taken to preserve the JSON 
file before invoking the delete command as  VFd  will  likely 
remove  it  from  the directory. The delete command syntax is 
similar to the add syntax illustrated  earlier  and  thus  is 
omitted. 
 
 
### Active configurations at VFd startup 
When   VFd   is   started,  it  makes  a  pass  through  the  
configuration directory and will _add_ all of the JSON  files 
which  it  finds  in  the directory. This allows the existing 
configuration to be reloaded easily when  VFd  is  restarted. 
VFd  will  ignore any files in the directory which **do not** 
have a .json suffix; from a manual testing  perspective  this 
allows  test  configuration files to be kept in the directory 
(e.g. VM1.x) and copied  into  the  desired  JSON  file  when 
needed. 
 
 
### Callback Command Hooks 
The  callback  command hooks (start_cb and stop_cb) in the VF 
configuration allow the _user_ to define a command  which  is 
executed  at the end of VFd startup initialisation (start_cb) 
and just prior to the normal shut down processing when VFd is 
terminated  (stop_cb).  The commands are executed as the user 
which owns  the  configuration  file,  and  the  command  may 
**not**  contain  an  embedded  semicolon  (;).  If  multiple 
commands are needed, then a script should be used. 
 
The exact use of these commands is unknown,  however  it  has 
been  observed  that  in some cases when VFd is restarted the 
ethernet driver in the VM fails  to  completely  reinitialise 
requiring  either  an  unload  and  reload  of the driver, or 
similar. One way to force a complete reinitialisation of  the 
VM's ethernet driver is to simulate a device removal followed 
by a device insertion which has shown to cause the driver  to 
initialise  to  the  point  that the VM can again communicate 
over the device after VFd has been stopped and started. 
 
 
# Logs and Debugging 
Log files are written to  the  directory  named  in  the  VFd 
config  file (generally in /var/log). VFd will create a dated 
log file (e.g. vfd.log.20160412) and the log will  be  rolled 
to  the  next  dated  version  with the first message written 
after 00:00Z. Log messages are dated with both a leading UNIX 
timestamp and human readable date; all times are in zulu (Z). 
The bracketed number following the date stamp  indicates  the 
log  level  of  the  message  (e.g.  _[2]_ indicates that the 
message is generated when the loging level is  set  to  2  or 
greater).  When  a  log  level  parameter is used on an iplex 
command, the more verbose output from VFd  can  be  found  in 
this log. 
 
 
 
## DPDK Log Messages 
Any  output  from  the  DPDK  library  is written to standard 
output (grumble) and is captured in the file vfd.std  in  the 
same  directory.  This  file  is  **not**  rolled,  and  with 
dpdk_loglevel settings less than 8 there is very  little,  if 
any, output. When things seem to behaving badly, set **both** 
of the dpdk log level settings to 8. 
 
 
 
## Service Start Messages 
The upstart script(s) will also add a log  file  which  might 
provide  useful  information should it appear that VFd is not 
starting properly. 
 
 
# Limitations And Requirements 
There are several limitations and/or requirements which  must 
be  observed  when  configuring both the PFs on a system, and 
with respect to the configuration of each VF within VFd.  The 
following  must  be  observed,  or  unexpected behaviour will 
result. 
 
 
## Number of VFs Per PF 
While it is possible to configure a small number of  VFs  for 
each  PF,  VFd requires that **all 32 VFs** be configured for 
each PF listed in the VFd configuration file. If  fewer  than 
32 Vfs are configured VFd is unable to properly see the queue 
ready (enabled) indicator and as a result the  necessary  NIC 
configuration  which must wait until the guest enables the Tx 
queues is never performed. Fewer than 32 VFs configured for a 
PF also interferes with VFd's ability to properly observe and 
report statistics about each VF (up/down state and counts). 
 
 
## MAC Limits 
The number of MAC addresses which may be defined  across  all 
VFs  for  a PF is limited by hardware to 128. When this limit 
is reached VFd will refuse to allow a VF configuration to  be 
added (the result of the iplex add command will be an error). 
Hardware also limits the number of MAC addresses which may be 
supplied for a single VF to 64. 
 
 
## VLAN ID Limits 
In  a  similar fashion to the MAC limits, the hardware limits 
the number of VLAN IDs which are configured across all VFs on 
a  single  PF to 64. VFd will enforce this limit by rejecting 
the attempt to add a VF configuration if the number  of  VLAN 
IDs  supplied  in  the  configuration  file  causes the total 
number of VLAN IDs defined for the PF to exceed this value. 
 
 
 
_____________________________________________________________
 
[1] A basic primer on SR-IOV can be found at 
http://www.intel.com/content/dam/doc/application-note/pci-sig-sr-iov-primer-sr-iov-technology-paper.pdf 
 
 
 
[2] All code fragments provided in this document assume 
Kshell; your mileage may vary with bash or other shells. 
 
*** 
 
 
**Source::** vfd_hackers.xfm 
 
**Original::** 12 April 2016 
 
**Revised::** 26 October 2016 
 
**Formatter::** tfm V2.1/0b086 
 
_____________________________________________________________
CAUTION: 
This document is generated  output  from  {X}fm  source.  Any 
modifications made to this file will certainly be overlaid by 
subsequent revisions.  The  proper  way  of  maintaining  the 
document  is  to  modify  the source file(s) in the repo, and 
generating the desired .pdf or .md versions of  the  document 
using {X}fm. 
_____________________________________________________________