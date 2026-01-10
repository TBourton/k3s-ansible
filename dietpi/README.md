# DietPI OS Install

## RPIs

For Raspberry PIs, follow <https://rpi4cluster.com/k3s-nodes/#dietpitxt>

## x86 Installer Image

For my random x86 machines I had lying around, I don't want to have the OS on a live images, since they have large HDDs. Hence we want to use the installer image instead of the live image. However, this makes two differences compared to the RPI version.

1. It's an ISO image - we can't copy the dietpi.txt config into the usb directly.
2. The storage needs partitioning

### Install DietPI

Install the iso >> dietpi-config

- ssh-server: openssh-server
- set static IP by copy DHCP assigned one
- autostart --> automatic login
- search software: 17,4,0,110,1,130
- security options --> CHange hostname

### Storage Partitioning

Use GParted live image <https://gparted.org/liveusb.php>. Shrink the boot partition to 32000MiB & then create a new partition with the remaining space.
