# Building a custom Kernel

The stock kernel distributed with Ubuntu 18.04 for Jetson devices does not come with proper dependences to support the Open vSwitch virtual switch that EdgeVPN.io builds upon.

While there are different ways to create a workflow to run Evio-enabled Jetson devices, such as cross-compilation and flashing a custom image, a simpler approach that works for development and testing is to boot up the Jetson device with the stock kernel/image, and build a new kernel that includes the proper dependences.

> **_NOTE:_** the process below has been tested on Jetson nano development kit devices. We have not tested other models of Jetson devices.

### Building the kernel

Once you have booted the device into the stock kernel, you need to follow the approach outlined in the nVidia documentation to build a custom kernel

You will need an nVidia developer account to be able to access kernel source files.

### Ensure your OS has proper dependences installed

    sudo apt update
    sudo apt install git-core
    sudo apt install build-essential bc

### Download and unpack kernel sources

You can login with your nVidia dev account and browse the downloads page, then select the latest “L4T Jetson Driver Package” to download

Copy the downloaded archive to your home directory on the nano, and expand it with:

    tar xf Tegra210_Linux_*.tbz2

### Sync gode with git repo

    cd Linux_for_Tegra
    ./source_sync.sh

When prompted, enter the git tag you want sync to, e.g. tegra-l4t-r32.5

You may be asked multiple times.

### Create baseline kernel build config file

Select a directory for your build, e.g. /home/username/build-ovs

    TEGRA_KERNEL_OUT=/home/username/build-ovs
    cd ~/Linux_for_Tegra/sources/kernel/kernel-4.9
    mkdir -p $TEGRA_KERNEL_OUT
    make ARCH=arm64 O=$TEGRA_KERNEL_OUT tegra_defconfig

### Edit .config to enable Open vSwitch modules

Now, you need to edit the .config file (with kernel build configuration parameters) to add Open vSwitch and GRE support as modules

    vi $TEGRA_KERNEL_OUT/.config 

Uncomment/add the following config entries:

    CONFIG_NET_IPGRE=m
    CONFIG_NET_IPGRE_DEMUX=m
    CONFIG_OPENVSWITCH=m
    CONFIG_OPENVSWITCH_GRE=m
    CONFIG_OPENVSWITCH_VXLAN=m

### Build the kernel

    make ARCH=arm64 O=$TEGRA_KERNEL_OUT -j4

You will be prompted about these (and perhaps other) options; enter your choices (N=no, m=module) manually:

    CONFIG_NET_MPLS_GSO=m
    CONFIG_MPLS_ROUTING=n
    CONFIG_PPTP=n

This will take more than an hour to complete.

### Copy kernel image and modules

Once the kernel is compiled, you need to copy the kernel image to /boot:

    sudo cp $TEGRA_KERNEL_OUT/arch/arm64/boot/Image /boot/Image-ovs

You also need to install and copy kernel modules:

    sudo make ARCH=arm64 O=$TEGRA_KERNEL_OUT modules_install \
    INSTALL_MOD_PATH=~/Linux_for_Tegra/rootfs/
    pushd ~/Linux_for_Tegra/rootfs
    sudo tar --owner root --group root -cjf kernel-ovs-modules.tbz2 lib/modules
    popd
    pushd /
    sudo tar -xf ~/Linux_for_Tegra/rootfs/kernel-ovs-modules.tbz2
    popd

### Edit boot config file

    Edit /boot/extlinux/extlinux.conf as follows to boot from the new kernel you just built as the primary option:

    TIMEOUT 30
    DEFAULT primary

    MENU TITLE L4T boot options

    LABEL primary
        MENU LABEL primary kernel
        LINUX /boot/Image-ovs
        INITRD /boot/initrd
        APPEND ${cbootargs} quiet root=/dev/mmcblk0p1 rw rootwait rootfstype=ext4 console=ttyS0,115200n8 console=tty0 fbcon=map:0 net.ifnames=0 

    # When testing a custom kernel, it is recommended that you create a backup of
    # the original kernel and add a new entry to this file so that the device can
    # fallback to the original kernel. To do this:
    #
    # 1, Make a backup of the original kernel
    #      sudo cp /boot/Image /boot/Image.backup
    #
    # 2, Copy your custom kernel into /boot/Image
    #
    # 3, Uncomment below menu setting lines for the original kernel
    #
    # 4, Reboot

    LABEL backup
        MENU LABEL backup kernel
        LINUX /boot/Image
        INITRD /boot/initrd
        APPEND ${cbootargs} quiet root=/dev/mmcblk0p1 rw rootwait rootfstype=ext4 console=ttyS0,115200n8 console=tty0 fbcon=map:0 net.ifnames=0
