## HOW TO BUILD THE UNIFIED BCM 57711/57810 KERNEL MODULE FOR VyOS (v1.3.0-rc1)
<b>Resulting md5sum: d4304df21a349a0a4f0b950c91590aa4 bnx2x.ko</b>

Updated for vyos-1.3.0-rc1-amd64.iso\
Using Debian Buster debian-10.8.0-amd64-netinst.iso as the base for the build

What is VyOS : https://vyos.readthedocs.io/en/latest/history.html

The sole objective of this post is to build the module

<b>Step 1:</b> Install VyOS and create an image from the console

I downloaded the VyOS snapshot release from here: https://vyos.net/get/snapshots/#vyos-1.3.0-rc1

see Install Vyos : https://vyos.readthedocs.io/en/latest/install.html (*Note: I didn't bother veryfing the signatures)

<b>Step 2:</b> Identify the target kernel version of the vyos distribution

    uname -a

example: Linux vyos 5.4.99-amd64-vyos #1 SMP Thu Feb 18 07:53:25 UTC 2021 x86_64 GNU/Linux

5.4.99 is the kernel version (this is important and is what you are looking to match)

You will need to checkout the appropriate repo commit for your target build

We will introduce the bnx2x module to the image at the end.

<b>Step 3:</b> Create a Debian Buster build environment on a VM or separate build machine. I used 96GB VM for the build

I downloaded Debian Buster debian-10.8.0-amd64-netinst.iso from here https://www.debian.org/releases/buster/debian-installer/

I suggest doing the install via regular text console and not via GUI (don't select graphical install).
Only enable ssh server and standard system utilities when selecting components.
why? because that will leave you in a known state.

<b>Step 4:</b> Enable root ssh access either by loging into console or ssh into the machine using the user account you created and then run the following as su

    su
    echo "PermitRootLogin yes" >> /etc/ssh/sshd_config
    /usr/sbin/service sshd restart

<b>Step 5:</b> log out then log back in as root via ssh

<b>Step 6:</b> The first thing that needs to be done is to modify /etc/apt/sources.list by running

    echo "deb http://deb.debian.org/debian buster-backports main contrib non-free" >> /etc/apt/sources.list
	
<b>Step 7:</b> Once you have modified the apt sources run the following to update and install the packages needed for the build (and a few extra)

    apt update
    apt -y install build-essential checkinstall fakeroot libncurses5-dev xz-utils libssl-dev bc flex libelf-dev bison curl ccache dpkg-dev debhelper git pkg-config uuid-dev firmware-bnx2x sudo live-build pbuilder devscripts python3-pystache python3-git python3-distutils rsync

<b>Step 8:</b> Get the vyos sources needed for the build

    cd ~
	git clone -b equuleus --single-branch https://github.com/vyos/vyos-build
	
<b>Step 9:</b> Checkout the commit with the matching kernel version

Go to: https://github.com/vyos/vyos-build/commits/equuleus and find the commit signature with the corresponding kernel, in our case 'commit f4be339392a75bda28a546547fe2ecf67660dda9'

    cd vyos-build
    git checkout f4be339392a75bda28a546547fe2ecf67660dda9

<b>Step 10:</b> Copy upnatom's unified patch to the vyos kernel patch directory 

    curl https://raw.githubusercontent.com/indigots/snippets/master/bnx2x/patches/git/bnx2x_warpcore_8727_2_5g_sgmii_txfault.patch -o ~/vyos-build/packages/linux-kernel/patches/kernel/bnx2x_warpcore_8727_2_5g_sgmii_txfault.patch

<b>Step 11:</b> Clone the stable branch of the linux kernel into the vyos build environment

    cd ~/vyos-build/packages/linux-kernel
	git clone git://git.kernel.org/pub/scm/linux/kernel/git/stable/linux-stable.git linux

<b>Step 12:</b> Checkout the required kernel version (see step 2)

    cd linux
    git checkout v5.4.99
    cd ..

<b>Step 13:</b> Build the kernel and bnx2x kernel module

    ./build-kernel.sh

<b>Step 14:</b> Copy the kernel and bnx2x.ko kernel module to home directory

    cp linux/drivers/net/ethernet/broadcom/bnx2x/bnx2x.ko ~/
    cp linux-image*.deb ~/

your modified kernel module can be found in the root home directory:\
~/bnx2x.ko

<b>Step 15:</b> Confirm wan/lan interface names, add a temporary IP to the lan interface, and enable ssh so the we can copy bnx2x.ko to the VyOS install

*Note: We wont touch the permissions on the VyOS install so some commands will need to be elevated using sudo

My future wan interface will be eth0 and my lan interface will be eth1

    ip link show
    sudo ip addr add 192.168.2.100/24 dev eth1
    config
    set service ssh port 22
    commit
    save

<b>Step 16:</b> copy bnx2x.ko to /home/vyos. I used winSCP but you could scp directly from the build machine to the host

<b>Step 17:</b> Update the image to use the new driver and reboot

login via ssh using user vyos

    sudo cp /home/vyos/bnx2x.ko /lib/modules/$(uname -r)/kernel/drivers/net/ethernet/broadcom/bnx2x/
    sudo /usr/sbin/update-initramfs -u -k all
    reboot

<b>Step 18:</b> From the console you can verify sync @ 2.5G before proceeding to configure VyOS by checking the future wan interface using ethtool

    ethtool eth0

Done !

Configure VyOS as desired (see https://docs.vyos.io/en/latest/quick-start.html)
