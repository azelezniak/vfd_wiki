The igb_uio driver is required and is a part of the DPDK library and is built by the DPDK build tools. It must be built for each system type as it relies on the kernel configuration when the kernel was built. The following are steps which should allow you to check out DPDK, build and install the driver

You'll need  git, make and gcc installed on the machine; sysadmin should be able to do that if they aren't there.  Test these with
gcc --version
git --version
make --version

If any one  is 'not found' ask the sysadmin to install. If you have root and can install then you can execute these directly:
        sudo apt-get install git
        sudo apt-get install gcc
        sudo apt-get install gmake



Once git/gcc are installed, these steps should work:

 1)      mkdir $HOME/build

 2)      cd $HOME/build

 3)      git clone http://dpdk.org/git/dpdk

 4)      cd dpdk

 5)      git checkout v16.11

 6)      make config T=x86_64-native-linuxapp-gcc
        (lots of build messages will scroll past)

 7)     tools/dpdk-setup.sh
        This will present a menu. select the option which is
        associated with  x86_64-native-linuxapp-gcc (usually 13-16
        but it varies from system to system depending on what is
        installed, so it's hard to say)
        Lots of messages will scroll past, finally it will
        announce that it is finshed; push return to go back to
        the main menu.

 8)      At the main menu, sleect the option to Insert IGB UIO module

 9)      return to the main menu and quit

