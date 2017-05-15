## Logged Error Messages
The following are all of the critical (CRI) and general (ERR) errors which are written to the VFd log file.  A brief explanation is given for each. 

**CRI: abort: no pciids were defined in the configuration file** <br />
This is a configuration error.  Who/whatever created /etc/vfd/vfd.cfg did 
not supply any PCIIDs which describe the PFs that VFd is to manage. PFs 
cannot be added/removed 'on the fly' and thus VFd refuses to run if none 
are defined.   The config file must be examined: if there are PCIIDs 
defined, then the config file should be sent to the development team for 
trouble shooting, and if there are none defined the problem lies with the 
installation.

**CRI: abort: unable to alloc memory for eal initialisation** <br />
The request to the DPDK library for memory failed.  If huge pages have 
been enabled (VFd normally does not require huge pages) then the issue is 
likely caused by a huge page shortage on the system. 

**CRI: unable to read configuration from \<name\>: \<system-err-text\>** <br />
The named configuration file was missing, ill formatted, or incomplete and 
thus could not be successfully parsed.  The source of the configuration 
file (Openstack usually) should be checked as the likely cause lies there.

**CRI: abort: unable to initialise request fifo** <br />
The request FIFO is usually created in /var/lib/vfd and the inability to 
initialise it is usually a permissions problem on the parent directory. 
VFd should be started as root, and if this message is issued VFd is likely 
being executed under another user and that will never result in success.
This message will generally be preceded by an ERR message that might have
more directly related information with respect to the root cause. 

**CRI: abort: unable to initialise dpdk eal environment** <br />
This is a general "catch all" message and there will be multiple DPDK 
messages in the /var/log/vfd/vfd.std (standard error) file that were 
generated directly from DPDK that indicate the problem. 

**CRI: abort: config file reports more devices than dpdk reports: cfg=\<number\> ndev=\<number\>** <br />
This message is generated when there is a configuration file error similar 
to the first error in this list.  The difference is that there are PCIIDs 
defined which the system isn't reporting to VFd.  Same actions as the for 
the the first error should be taken.

**CRI: abort: cannot create refresh_queue thread** <br />
This is an internal error and should be referred to the development team 
for resolution.

**CRI: abort: mbfuf pool creation failed** <br />
The request to the DPDK library for memory failed.  If huge pages have 
been enabled (VFd normally does not require huge pages) then the issue is 
likely caused by a huge page shortage on the system. 


**CRI: abort: port >= rte_eth_dev_count** <br />
This error results when an attempt is made to access or configure a device
(port) using an ID which is out of range.  The dev team should be contacted
if this error is generated.

**CRI: abort: port initialisation failed: \<number\>** <br />
**CRI: abort: unable to initialise nic with base config:** <br />
**CRI: abort: cannot configure port \<number\>, retval \<number\>** <br />
**CRI: abort: cannot setup rx queue, port \<number\>** <br />
**CRI: abort: cannot setup tx queue, port \<number\>** <br />
**CRI: abort: cannot start port \<number\>** <br />
These are all device related failures. They may indicate faulty hardware or
issues with the driver being used to manage the PF (should be igb_uio, but
others have been used).   The standard error file (usually /var/log/vfd/vfd.std)
should be checked for related DPDK generated messages which may lead to the 
actual cause of the problem. 

**ERR:  the last element of argc wasn't nil** <br />
This is an internal error, but is not fatal.  VFd deals with it by adding a nil 
argument and continuing.   The current configuration file, and VFd log, should
be supplied to the development team as it is another VFd component which is
generating the faulty array and the code requires investigation. 

**ERR: unable to create request fifo (\<name or string\>): \<name or string\>** <br />
This message is generated at the time that the FIFO cannot be initialised and 
will likely contain the related system error message as the final \<string\>.
VFd may attempt several times to create the request FIFO, and thus this error
is only significant if VFd aborts due to the FIFO initialisation failure (and
generates the previously described critical message).

**ERR: failed to create a json parsing object for: \<name or string\>** <br />
This message is generated when a configuration file has incorrectly formatted 
json.  Depending on the parsing error details may be provided. \<name\> on this 
error should be the configuration file being parsed.  

**ERR: request received without action: \<name or string\>** <br />
**ERR: unrecognised action in request: \<name or string\>** <br />
These messages indicated that a  request passed to VFd from iplex was incomplete.  
This generally indicates an error with iplex and should be referred to the 
development team. 

**ERR: memory allocation error tying to alloc request for: <name or string>** <br />
This error is generated at the site in the code where the memory allocation 
failed and may contain a more detailed system error message.  VFd may attempt
to allocate the memory several times before aborting, thus this message is only
significant if the related critical error message (previously discussed) is 
also generated.

**ERR: gen_stats: realloc failed** <br />
This error indicates that VFd is unable to allocate memory in order to fulfill
a sats request.  If several of these occur over a short period of time, the 
log file should be made available to the development team for examination. 

**ERR: bad unicast hash table parameter, return code = \<number\>** <br />
This error is generated during overall initialisation and indicates that the 
DPDK library creation of a hash was not successful.  The return code can be 
used to determine (via the DPDK documentation) what the cause of the problem
was.   

