---
title: Open-VM-Tools Arch-Linux Architecture ARM64 For Mac M1 (Arch.Linux.Arm--aarch64)
date: 2023-12-02
tags:
  - "Open-VM"
  - "Linux"
  - "arm64"
  - "Arch Linux"
categories:
  - "Pentesting"
thumbnail:
  src: "https://lh3.googleusercontent.com/drive-viewer/AK7aPaCpGf0VLCjTYRuF1D65CYCXtySu64GKiVhgrorwvxG6UpPcUCafjdwsS7Wk4wkw-qEZBT9kr21u66-BzkQlh0qKDvEwcg=s1600"
  
---


Open VM Tools (open-vm-tools) is the open source implementation of VMware Tools for Linux and FreeBSD guest operating systems. The open-vm-tools suite is bundled with some Linux operating systems and is installed as a part of the OS, eliminating the need to separately install the suite on guest operating systems. All leading Linux vendors support the open-vm-tools suite on vSphere, Workstation, and Fusion, and bundle open-vm-tools with their product releases. For information about OS compatibility check for the open-vm-tools suite, see the VMware Compatibility Guide at [resources](http://www.vmware.com/resources/compatibility).

<!--more-->



# open-vm-tools

For better managing guest operating systems, the open-vm-tools suite includes the following packages:

* The core open-vm-tools package contains the core open-vm-tools user space utilities, application programs, and libraries, including vmtoolsd, to help effectively manage communication between your host and guest OSs. This package includes features as, synchronizing guest OS clocks with the virtualization platform, transferring files between hosts and guests, sending heartbeat information from guest OSs to the virtualization infrastructure to support vSphere High Availability (HA), publishing resource utilization and networking information of the guest OSs to the virtualization platform, and so on.

* The open-vm-tools-desktop package is optional and includes additional user programs and libraries to improve the interactive functionality of desktop operations of your virtual machines. The package enables you to resize a guest display to match its host console window or the VMware Remote Console Window for vSphere. The package also allows you to copy and paste between host and guest OSs, as well as to drag and drop between guests and a host for the VMware Workstation and VMware Fusion products.

* The open-vm-tools-devel package contains libraries and additional documentation for developing vmtoolsd plug-ins and applications.

It's important to note that adding the Kali Linux repository to a non-Kali Linux distribution may have unintended consequences and can potentially cause conflicts with existing software packages. Users should exercise caution and thoroughly understand the implications before adding external repositories.

### Problem

First, we have to check whether open-vm-tools has been installed on VMware Arch Linux or not, apart from trying to drag and drop between the Mac host and Arch, we can see it with the command:

```bash
sudo systemctl status vmtoolsd.service
sudo systemctl status vmware-vmblock-fuse.service
```

The average user definitely won't find anything from the output results that we did earlier, that's because by default open-vm-tools on Arch Linux (especially for DWM, I3WM, BSPWM and other users) is required to manually compile the configuration, actually it's very easy This is done because the package is already in AUR, see [here](https://archlinux.org/packages/extra/x86_64/open-vm-tools/).

I just spent a lot of time, research, reading this [documentation](https://docs.vmware.com/en/VMware-Tools/12.3.0/com.vmware.vsphere.vmwaretools.doc/GUID-8B6EA5B7-453B-48AA-92E5-DB7F061341D1.html). but that didn't work.

However, the arm architecture was not found, or has not been released, we can see [here](https://archlinuxarm.org/forum/viewtopic.php?f=5&t=15763). This is just a proposal and there has been no follow-up from the developer.

Therefore, I developed several experiments, failures, to help and solve this problem.

I really understand the difficulties of not having open-vm-tools, apart from the screen not being perfect, typing experiencing bugs, plus we can't easily copy and paste between arch and mac, that's very torturous, isn't it?

### Solution

first of all, we need to clone the repository, because i assume that you can’t access copy paste between mac and arch, we need manually write, or you can make it with ssh inside mac.

```bash
sudo ssh arch@182.927.22
```

(Change arch with your arch username, and ip you can found with `ip addr`)Then enter your arch linux password

Next, here we go:

```bash
git clone https://github.com/0xZipp0/vm-tools.git
```

Please make the tools automation script is executed with:

```bash
chmod +x build.sh
```

We can see all of the dependencies and tools need here, 

```bash
➜  Documents tree vm-tools
vm-tools
├── README.md
├── build.sh
├── vmtoolsd.service
├── vmware-user.desktop
└── vmware-vmblock-fuse.service

1 directory, 5 files
```

Next, we go with command:

```bash
cd vm-tools
sudo ./build.sh
or
./build.sh
```

Inilah layout atau tampilan dari build.sh, kita bisa melihat beberapa fungsi dan instalasi beberapa base yang berguna untuk proses open-vm-tools

```bash
#!/usr/bin/env bash
set -euo pipefail

scriptDir() {
	P=`pwd`
	D="$(dirname $0)"
	if [[ $D == /* ]]; then
		echo $D
	elif [[ $D == \.* ]]; then
		J=`echo "$D" | sed 's/.//'`
		echo "${P}$J"
	else
		echo "${P}/$D"
	fi
}
SD=`scriptDir`
CUR=`pwd`
P="$HOME"
cd $P

sudo pacman -Sy fuse2 icu iproute2 libdnet libmspack libsigc++ \
         libxcrypt libcrypt.so libxss lsb-release procps-ng \
         uriparser gdk-pixbuf-xlib chrpath doxygen gtkmm3 libxtst \
		 python rpcsvc-proto netctl networkmanager cunit \
		 --noconfirm

VER="${1:-}"
VMURL="https://github.com/vmware/open-vm-tools.git"
if [ ! -z $VER ]; then
	VMURL="https://github.com/vmware/open-vm-tools.git --branch=$VER"
fi
git clone $VMURL
cd open-vm-tools/open-vm-tools
autoreconf -i
./configure \
    --prefix=/usr \
    --sbindir=/usr/bin \
    --sysconfdir=/etc \
    --with-udev-rules-dir=/usr/lib/udev/rules.d \
    --without-xmlsecurity \
    --without-kernel-modules
make
make check
sudo make install
sudo ldconfig

cd $SD
sudo cp ./vmtoolsd.service /usr/lib/systemd/system/vmtoolsd.service

sudo cp ./vmware-vmblock-fuse.service \
	/usr/lib/systemd/system/vmware-vmblock-fuse.service

# sudo cp ./vmware-user.desktop /etc/xdg/autostart/vmware-user.desktop

sudo systemctl enable vmtoolsd.service
sudo systemctl enable vmware-vmblock-fuse.service

sudo pacman -R chrpath doxygen rpcsvc-proto cunit --noconfirm
echo "vmhgfs-fuse /mnt/hgfs  fuse defaults,allow_other   0   0" | sudo tee -a /etc/fstab

cd $CUR
rm -rf $P/open-vm-tools
```

### Script Explanation

The script is a bash script used to download, configure, build, and install the open-vm-tools software from the GitHub repository. Here's a brief and concise explanation of the steps performed by this script:

1. The script is written in the Bash shell scripting language.
2. The script sets shell options (`set -euo pipefail`) to handle errors and failures during execution.
3. The `scriptDir()` function retrieves the current script's directory.
4. Variables `SD` and `CUR` store the script's directory and the current working directory, respectively.
5. The script changes the directory to the user's home directory.
6. The required packages for compiling open-vm-tools are installed using the `pacman` package manager.
7. The script checks if a specific version is provided as an argument and updates the repository URL accordingly.
8. The script clones the open-vm-tools repository from the specified URL.
9. The script navigates to the downloaded repository directory.
10. Configuration and build steps are performed using `autoreconf`, `./configure`, `make`, and `make check`.
11. The built open-vm-tools are installed using `sudo make install`.
12. The system's library cache is updated using `sudo ldconfig`.
13. The relevant service files are copied to the appropriate directories.
14. The services `vmtoolsd` and `vmware-vmblock-fuse` are enabled to start automatically on system startup.
15. Unnecessary packages are removed using `pacman -R`.
16. A configuration line is added to the `/etc/fstab` file to associate the HGFS file system with `/mnt/hgfs`.
17. The script returns to the original working directory and removes the downloaded open-vm-tools directory.

In summary, the script performs all the necessary steps to download, configure, build, and install open-vm-tools from the GitHub repository.