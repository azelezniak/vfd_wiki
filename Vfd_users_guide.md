 
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
 
Under normal circumstances VFd, and any related software tools, 
are installed on the system via some distribution mechanism (chef, 
puppet, or the like), which is also responsible for the creation 
of the configuration file in /etc and ensures that the VFs have 
been created. Nova, or some similar VM management environment is 
generally responsible for creating the individual VF 
configurations, placing them into the /var directory, and using 
iplex to add the configuration information as a VM is created. 
 
At times it may be necessary to install and make use of VFd 
without the usual distribution and virtual environment management 
components. This guide provides a way to hack[2] together such a 
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
  
2 Bind the PF to the igb_uio driver (if not already bound) 
  
3 Disable all current VFs (echo 0 >max_vfs) 
  
4 Enable the number of VFs desired (echo 32 >max_vfs) 
  
5 Verify that the VFs were added (ls -al in the directory and 
 expect to find symbolic links like virtfn3->../0000) 
 
 
 
 
The  above  example shows the creation of 32 VFs; up to 32 VFs may 
be added to a single PF. As a final step, the VFs should be  bound 
to  the  vfio-pci driver if they were not automatically bound when 
created (in our lab they are automatically bound, so this step may 
not be needed). The dpdk_nic_bind utility (installed with VFd) can 
be used to do this as illustrated in figure 1. 
 
     
     
     
           dpdk_nic_bind -b vfio-pci 0000:01:10.7
 
 
 
Figure 2: Command used to bind the vfio-pci driver to a VF. 
 
Note that only the PCI address is needed for these  commands,  and 
if  another  driver  is  bound  to the VF it will automatically be 
unbound first. 
 
 
### Number of VFs to create 
A logical question at this point would be _how many VFs  should  I 
create?_  The answer depends on whether or not VFd will be running 
with quality of service (QoS) enabled. When running in  QoS  mode, 
it  is  a  requirement  that  31  VFs be created for each PF. When 
running in _version 1 mode_ (QoS  disabled),  it  is  possible  to 
allocate  less  than  32  VFs  for  a PF, but it is recommended to 
allocate 32. 
 
 
## PF/VF Verification 
The dpdk_nic_bind utility, dpdk-devbind in releases  after  16.11, 
can  also  be  used  to  verify  the  state of the various network 
interfaces on the system. Executing with the --status option  will 
cause  it  to  list all network interfaces. If the VFs were added, 
and bound to the correct  driver,  the  output  from  the  command 
should list them all under a heading which indicates that they are 
bound to a DPDK capable driver. Figure 2 illustrates the  expected 
output  which  lists  the  PFs bound to the uio driver and the VFs 
bound to the vfio-pci driver. 
     
     
     
         
         root@agave207:/var/log# dpdk_nic_bind --status
         
         Network devices using DPDK-compatible driver
         ============================================
         0000:08:00.0 'Ethernet 10G 2P X520 Adapter' drv=igb_uio unused=vfio-pci
         0000:08:00.1 'Ethernet 10G 2P X520 Adapter' drv=igb_uio unused=vfio-pci
         0000:08:10.0 '82599 Ethernet Controller Virtual Function' drv=vfio-pci unused=igb_uio
         0000:08:10.1 '82599 Ethernet Controller Virtual Function' drv=vfio-pci unused=igb_uio
         0000:08:10.2 '82599 Ethernet Controller Virtual Function' drv=vfio-pci unused=igb_uio
         0000:08:10.7 '82599 Ethernet Controller Virtual Function' drv=vfio-pci unused=igb_uio
         :
         :
         0000:08:17.5 '82599 Ethernet Controller Virtual Function' drv=vfio-pci unused=igb_uio
         
         Network devices using kernel driver
         ===================================
         0000:02:00.0 'NetXtreme BCM5720 Gigabit Ethernet PCIe' if=em1 drv=tg3 unused=igb_uio,vfio-pci *Active*
         0000:02:00.1 'NetXtreme BCM5720 Gigabit Ethernet PCIe' if=em2 drv=tg3 unused=igb_uio,vfio-pci 
         
         Other network devices
         =====================
         <none>
         
 
 
 
Figure 3: Partial output generated by  the  dpdk_nic_bind  utility 
with the --status option. 
 
 
 
## IOMMU Groups 
If  a  NIC  has  two  physical  interfaces,  these will be grouped 
together from an IOMMU perspective, and in order for VFd  to  have 
access to the NICs, all of the devices in a group must be bound to 
a DPDK capable driver. If all devices belonging to a group are not 
bound  to  the  DPDK capable driver, an error will be generated as 
VFd attempts to initialise, and the DPDK library will abort. 
 
Given a known PF, it is simple to determine  which  other  devices 
also  belong  to  the  same  group. Figure 3 shows a simple set of 
commands[3] which can be used to find the location of a known  PCI 
device  and  then  all other devices in the same group. The output 
from the command shows the group number as the last  directory  in 
the  path which is the group number. A subsequent list command can 
then be used to list all devices in that group. 
 
     
     
     
           find sys/kernel/iommu_groups -name 0000:01:00.1 | read p junk
           ls ${p%/*}
 
 
 
Figure 4: Finding iommu group members. 
 
 
 
# Installation And Configuration 
VFd is distributed in a .deb package and can be installed  with  a 
simple   dpkg  command.  The  installation  process  creates  the  
configuration directory in /etc  (adding  a  sample  configuration 
file),  creates  the  necessary  directories  in  the /var/lib and 
/var/log directories, and places  the  following  programmes  into 
/usr/bin 
 
 
**vfd:** The daemon itself (64 bit Linux binary) 
 
**iplex:**  The  command  line  tool  used to communicate with VFd 
(python 2.x) 
 
**vfd_pre_start:** The script which manages  the  startup  process 
(python) 
 
**dpdk_nic_bind:**  An  opensource programme which facilitates the 
binding and unbinding of  drivers  to/from  virtual  and  physical 
functions (python) 
 
 
 
 
The  installation  process  also  installs the necessary _upstart_ 
mechanism allowing the familiar _service_ command to  be  used  to 
start and stop VFd. 
 
 
## Configuration 
The  configuration  file,  vfd.conf,  must  be created in /etc/vfd 
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
 
**log levels:** The verbosity of running chatter  emitted  by 
VFd  can  be  controlled  by these settings. Four options are 
provided which control the chattiness  during  initialisation 
(usually  more  information  is  desired)  and  a level which 
affects the drivel  after  initialisation  is  complete.  Log 
levels  are  supplied  for  both  VFd proper and for the DPDK 
library functions as it is useful to control them separately. 
 
**config_dir:** This is the directory where the individual VF 
configuration files are expected to be placed. The default is 
shown in the example. 
 
**cpu_mask:** This is a single bit which indicates which  CPU 
the  DPDK  library will associate with. VFd limits this value 
to a single bit, and odd results may  occur  if  a  multi-bit 
integer is given. 
 
**pciids:** Explained in the following section 
 
 
 
 
 
 
### Pciid Array 
The  array  of  pciids  supplied  in  the  configuration file 
defines the PFs which VFd will attempt to manage. In addition 
to the PCI address, the following value should be supplied: 
 
 
**mtu:**  The  mtu  value  for  the  device. If omitted the _ 
default_mtu_ value described earlier is used. 
 
**enable_loopback:** When set to true allows the NIC  to  act 
as  a  bridge  and permits traffic from one guest to another, 
hosted on the same PF, to be  looped  directly  back  without 
going out on the physical wire. The default value is false. 
 
 
 
 
 
 
### Comments And Other Elements 
Any  field  which VFd doesn't expect is ignored, and thus the 
comments   array   allows   the   creator  to  document  the  
configuration  without  interfering  with  VFd  Order  of the 
fields is unimportant,  however  the  field  names  are  case 
sensitive. 
 
 
## QoS Configuration 
With  version  2,  VFd  supports  a  QoS (quality of service) 
allowing the system manager to define the traffic  class  and 
queue parameters which are then applied to the underlying NIC 
by VFd.  These  parameters  exist  as  overall  configuration 
information  which  is  defined in the main VFd configuration 
file, and information which is specific to each VF  and  thus 
supplied in the configuration file for each VF. 
 
The  parameters  allow  QoS to be enabled, and define each of 
the four supported traffic classes. In addition  to  separate 
traffic  classes,  it is possible to group two or more of the 
classes into what are known as bandwidth groups. The  set  of 
parameters  for QoS can be quite lengthy, and thus a complete 
example is presented as an appendix to  this  document  as  a 
part   of  an  overall  VFd  configuration  file.  Figure  5  
illustrates how each traffic class may be defined to VFd. 
 
The traffic class definition object  contains  the  following 
fields: 
 
 
**name :** The traffic class name (future and/or diagnostic 
output). 
 
**pri :** The priority of the traffic class (0 - 3  where  0 
is the lowest). 
 
**lsp :** When true, sets link strict priority on the class. 
 
**bst :** When true, sets bandwidth group strict priority on 
the class. 
 
**max_bw :** Maximum amount of  bandwidth  as  a  percentage 
(future). 
 
**min_bw  :**  Minimum  amount of bandwidth as a percentage 
(future). 
 
 
 
 
 
### Traffic Class Over Subscription 
By default, the percentages associated with any traffic class 
queue sharing (please refer to the section on configuring VFs 
for more details) must not exceed 100%. When a request to add 
a  VF  is  made, VFd will examine the queue share settings in 
the VF configuration file and will reject the request if  the 
new settings cause one or more of the queue percentage totals 
to exceed 100%. As an  example,  if  two  VFs  are  currently 
managed  by  VFd  and  each  has  been given 50% share of the 
priority 2 queue, it will be impossible  to  add  another  VF 
that defines a non-zero share for the priority 2 queue. 
 
_Over  subscription_ allows for the sum of percentages of any 
traffic  class  queue  to  exceed  100%.  When  enabled,  the 
percentages  for  each  queue with a total exceeding 100% are 
normalised such that the values given to the NIC total  100%. 
Continuing the previous example, if a third VF is added, with 
a queue share of 20%, taking the total  for  the  priority  2 
queue  to  120%, the actual percentages used when configuring 
the underlying hardware would be reduced to approximately 83% 
of the indicated value (41.5%, 41.5% and 16.6%). 
 
To  enable over subscription for a PF, he following field may 
be added to any  of  the  definitions  in  the  pciid  array. 
(Please see Appendix A for a complete example.) 
 
     
     
     
           "vf_oversubscription": true,
 
 
 
 
IF  the set VF configuration files define one or more traffic 
classes which do not total 100%, the values  are  'normalised 
up' such that the overall usage is equal to 100%. 
 
 
### Enabling QoS 
By  default, VFd starts up in non-QoS mode (v1 mode) and thus 
QoS must be enabled in the main configuration file  in  order 
for  VFd  to  recognise  any  of  the  QoS configuration. The 
following configuration file  parameter  may  be  defined  to 
enable QoS: 
 
     
     
     
           "enable_qos": true
 
 
 
 
The  -q  command  line option can also be used to turn on QoS 
which may be a more convenient way of enabling  this  feature 
when testing. 
 
 
# Running VFd 
The  daemon  can  be  started  manually, or using the upstart 
start and stop commands as would be done for any other system 
service.  When  running manually the user must be _root,_ and 
various command line options may be provided allowing  for  a 
diverse testing environment. The VFd command line syntax is: 
 
     
     
     
           vfd [-f] [-n] [-p config-parm-file] [-q]
 
 
 
 
Where: 
 
 
**-f:** Causes VFd to run without detaching from the tty 
(foreground) 
 
**-n:** Enables _no-execution_ mode which prevents VFd from 
actually doing anything through DPDK 
 
**-p:** Provides the filename of the configuration file 
(default is /etc/vfd/vfd.cfg) 
 
**-q:** Enable QoS. 
 
 
 
 
 
 
[ IMAGE ] 
 
 
The  illustration  in  figure 5 shows the relationship of the 
daemon with the iplex utility, configuration files,  and  the 
assumed interaction with nova. 
 
 
## VF Configuration 
In  a  normal  operation  environment  nova,  or some similar 
virtualisation manager, would create a configuration file for 
each  VM and place that file into the configuration directory 
named in the main VFd config file. When manually managing the 
environment,  the  hacker  is  responsible for creating these 
files, adding them to the config  directory,  and  using  the 
iplex  command  line tool to cause VFd to parse and configure 
the NIC(s) based on the file contents. The following sections 
describe  this  process  and  provide  the  details of the VF 
configuration files. 
 
 
### Internal configuration management 
VFd manages only the VFs which have been added to its view of 
the  world;  if  a  configuration  file is not provided for a 
PF/VF combination, VFd makes no attempt to  manage  that  VF. 
This  allows  for  VFs  to  be  managed  and/or used by other 
elements on the system which do not need VFd supervision. VFd 
will   continue   to  manage  (configure)  a  VF  until  the  
configuration is deleted with an iplex command. 
 
 
### The VF configuration file 
The VF configuration file is a simple  set  of  JSON  encoded 
parameters  which defines the aspects of a single VF that VFd 
is   expected  to  manage.  These  files  are  typically  in  
/var/lib/vfd/config,  however the directory where VFd expects 
to find them may be set in the main VFd configuration file. 
     
     
     
         {
           "comments": [
             " configure VF 1 on the pciid given",
             " comments are ignored"
           ],
          
           "name":             "VM_daniels1/uuid-dummy-daniels-1",
           "pciid":            "9999:00:00.0",
           "vfid":             1,
           "strip_stag":       false,
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
 
 
 
Figure 6: Sample VF configuration file  normally  created  by 
nova. 
 
Figure  5  provides a sample configuration which contains the 
following information: 
 
 
**name:**   Is   any   string   provided  by  the  user  for  
identification of the configuration file. 
 
**pciid:**  is  the ID of the **PF** that the VF is allocated 
from. 
 
**vfid:** Is the ID  number  (0-31)  of  the  VF  on  the  PF 
supplied by the pciid. 
 
**vlans:** Is an array of one or more VLAN IDs which VFd will 
configure on the VF. When more than one VLAN ID is  supplied, 
the  strip_stag  field  **  must ** be set to _false_ as when 
there are multiple IDs supplied it is impossible for the  NIC 
to know which ID to insert on outbound traffic. 
 
**macs:** Is an array of one or more MAC addresses which VFd 
will configure on the VF. Any MAC addresses placed  into  the
array  act  as a whitelist and outbound (Tx) packets from the
guest with any listed MAC address as the source will  not  be
blocked.  The MAC address assigned to the VF is automatically
included in the list by VFd, and thus does ** not ** need  to
be supplied, therefore it is valid to have an empty MAC list.


Is an array of one or more MAC addresses which VFd 
will configure on the VF. It is generally  **not**  necessary 
to configure any MAC addresses. 
 
**start_cb:** Is a command (script) which VFd will execute as 
the last step in the VFd start up process.  (See  section  on 
callback command hooks below.) 
 
**stop_cb:**  Is a command (script) which VFd will execute as 
the first step in the VFd shutdown process. 
 
 
 
 
 
 
### Strip/Insert VLAN IDs 
The VF may be configured to automatically remove the VLAN tag 
as  each  packet  is  received, and to automatically insert a 
VLAN tag as each packet is transmitted.  Both  functions  are 
set  with  the same strip_stag field; as the strip and insert 
operations are related,  they  are  both  either  enabled  or 
disabled  and  thus  only  one  field is necessary. When this 
field is set to true, the array of VLAN IDs must contain only 
a  single  ID  as  it is impossible for the NIC to know which 
VLAN ID to insert if multiple IDs are defined in  the  array. 
Thus,  VFd  will  reject  the configuration if the strip stag 
field is set to true, and the VLAN array has  more  than  one 
element. 
 
 
### VF Quality of service configuration 
In  addition  to  the  parameters  noted  above, a set of QoS 
parameters may be supplied  in  the  VF  configuration  file. 
These  parameters, defined with the queues array, are ignored 
when VFd is not started in QoS mode, and are used  to  define 
the  amount  of  bandwidth  that  the  VF  should receive for 
outbound packets. The queues parameters  are  illustrated  in 
figure 6. 
 
     
     
     
          "queues": [
            { "priority": 0, "share": "10%" },
            { "priority": 1, "share": "14%" },
            { "priority": 2, "share": "20%" },
            { "priority": 3, "share": "10%" },
           ]
 
 
 
Figure 7: Quality of service parameters which may be supplied 
in a VF configuration file. 
 
 
Each element in the QoS _queues_ array defines the  bandwidth 
share  for  each  of  the  four  traffic  classes  that  are  
supported. Traffic classes are numbered from 0 (low priority) 
to  3  (highest  priority). In the illustration, the VF being 
configured would be allocated 20% of the priority  2  traffic 
class. 
 
 
### Adding a configuration 
The  VF  configuration  files  are  placed into the directory 
named in  the  main  VFd  configuration  file,  generally  in 
/var/lib/vfd/config and are named with any prefix and a .json 
suffix.  Once  a  configuration  file  is  placed  into  the  
directory,  the  iplex  command  line tool is used to add the 
configuration to  VFd.  The  overall  syntax  for  the  iplex 
command  is  given  in  figure  7,  with an example of an add 
command, to add VM1.json, is illustrated in figure 8. 
 
     
     
     
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
 
 
 
Figure 8: The usage output from the iplex command. 
 
     
     
     
           iplex add VM1
 
 
 
Figure 9: Sample iplex command to add a file  named  VM1.json 
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
When running with QoS enabled, VFd requires that 31 VFs (yes, 
31,  that is not a typo) be created for each PF. This ensures 
that the proper number of traffic classes  are  created,  and 
allows one set of queues for the PF to 'own.' 
 
When  running  with  QoS  disabled,  it should be possible to 
configure any number of VFs from 1 through 32. 
 
 
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
 
 
## Kernel Version 
A kernel version of 3.13.0-60 or later is required. VFd  will 
likely run on earlier kernels, but there have been some which 
presented very odd packet  dropping  behaviour  with  earlier 
versions. 
 
 
# Notes 
 
 
 
_____________________________________________________________
 
[1] A basic primer on SR-IOV can be found at 
//www.intel.com/content/dam/doc/application-note/pci-sig-sr-iov-primer-sr-iov-technology-paper.pdf 
This link was 'scrubbed' as it might be causing account flagging when the page is replaced. 
 
 
 
 
[2] We use the terms hack, hacker, and hacking in their 
original computer science sense. Hacking is the act of 
manually manipulating a programme or environment in order 
to tailor it to the programmer's needs. In a production 
environment, the overall VFd configuration, and the 
specifications of each underlying VF, will likely be 
managed by an automated process, however in a test 
environment it will be necessary for the systems programmer 
and/or tester to hack together the various configurations 
and to execute the necessary configuration commands to 
interact with VFd. 
 
 
 
[3] All code fragments provided in this document assume 
Kshell; your mileage may vary with bash or other shells. 
 
*** 
 
 
# Appendix A -- Configuration File Examples 
 
 
## Main VFd Configuration file 
     
     
     
         
         {   
             "comment":      "config for agave207",
             "log_dir":      "/var/log/vfd",
             "log_keep":     9,
             "log_level":    3,
             "init_log_level": 2,
             "config_dir":   "/var/lib/vfd/config",
             "fifo":   "/var/lib/vfd/request",
             "cpu_mask":   "0x01",
             "dpdk_log_level": 2,
             "dpdk_init_log_level": 8,
             "default_mtu": 9000,
             "enable_qos": true,
         
             "pciids": [ 
              {       "id": "0000:08:00.0",
               "mtu": 9000,
               "enable_loopback": false,
               "pf_driver": "igb-uio",
               "vf_driver": "vfio-pci",
               "vf_oversubscription": true,
         
               "tc_comment": "traffic classes define human readable name, tc number (priority) and other parms",
               "tclasses": [
                 {
                   "name": "best effort",
                   "pri": 0,
                   "llatency": false,
                   "lsp": false,
                   "bsp": false,
                   "max_bw": 100,
                   "min_bw": 10
                 },
                 {
                   "name": "realtime",
                   "pri": 1,
                   "llatency": false,
                   "lsp": false,
                   "bsp": false,
                   "max_bw": 100,
                   "min_bw": 40
                 },
                 {
                   "name": "voice",
                   "pri": 2,
                   "llatency": false,
                   "lsp": false,
                   "bsp": false,
                   "max_bw": 100,
                   "min_bw": 40
                 },
                 {
                   "name": "control",
                   "pri": 3,
                   "llatency": false,
                   "lsp": false,
                   "bsp": false,
                   "max_bw": 100,
                   "min_bw": 10
                 }
               ],
               "bwg_comment": "groups traffic classes together, min derived from TC values",
               "bw_grps":
               {
                 "bwg0": [0],
                 "bwg1": [1, 2],
                 "bwg2": [3]
               }
             },
         
             {       "id": "0000:08:00.1",
               "mtu": 9000,
               "enable_loopback": false,
               "pf_driver": "igb-uio",
               "vf_driver": "vfio-pci",
               "vf_oversubscription": false,
               "tclasses": [
                 {
                   "name": "best effort",
                   "pri": 0,
                   "llatency": false,
                   "lsp": false,
                   "bsp": false,
                   "max_bw": 100,
                   "min_bw": 10
                 },
                 {
                   "name": "realtime",
                   "pri": 1,
                   "llatency": false,
                   "lsp": false,
                   "bsp": false,
                   "max_bw": 100,
                   "min_bw": 40
                 },
                 {
                   "name": "voice",
                   "pri": 2,
                   "llatency": false,
                   "lsp": false,
                   "bsp": false,
                   "max_bw": 100,
                   "min_bw": 40
                 },
                 {
                   "name": "control",
                   "pri": 3,
                   "llatency": false,
                   "lsp": false,
                   "bsp": false,
                   "max_bw": 100,
                   "min_bw": 10
                 }
               ],
               "bw_grps":
               {
                 "bwg0": [0],
                 "bwg1": [1, 2],
                 "bwg2": [3]
               }
             }
             ]
         }
 
 
 
 
 
## VF Configuration file 
     
     
     
          {
             "comments":      "",
         
             "name":       "daniels_0_1",
             "pciid":      "0000:08:00.0",
             "vfid":       2,
             "strip_stag":       true,
             "allow_bcast":      true,
             "allow_mcast":      true,
             "allow_un_ucast":   true,
             "vlan_anti_spoof":  true,
             "mac_anti_spoof":   true,
             "vlans":      [ 10 ],
             "macs":       [ ],
         
             "queues": [
           { "priority": 0, "share": "10%" },
           { "priority": 1, "share": "10%" },
           { "priority": 2, "share": "10%" },
           { "priority": 3, "share": "10%" },
             ]
          }
 
 
 
 
 
 
 
 
 
_____________________________________________________________
_____________________________________________________________
CAUTION: 
This  document  is  generated  output  from {X}fm source. Any 
modifications made to this file will certainly be overlaid by 
subsequent  revisions.  The  proper  way  of  maintaining the 
document is to modify the source file(s)  in  the  repo,  and 
generating  the  desired .pdf or .md versions of the document 
using {X}fm. 
_____________________________________________________________
 
**Source:** vfd_hackers.xfm 
 
**Original:** 12 April 2016 
 
**Revised:** 23 January 2017 
 
**Formatter:** tfm V2.1/0b086 
 
