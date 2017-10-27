# hetzner-proxmox-pfsense

I did not really like the NAT solutions recommended for ESXi/Proxmox/SmartOS on Hetzner.
The perfectionist in me wanted to have the hypervisor behind the same firewall as the VM's.
This is how I managed to implement pfSense with 1 NIC (1 IP) in Proxmox using PCI passthrough.

## step 1: install proxmox
Request a LARA (their nickname for KVM/IPMI) session.
I had an installed flash drive after trying SmartOS, so I just dd'ed the .iso from their live linux rescue system.
Either that or request them to mount the iso in the LARA (KVM/IPMI) session.

## step 2: enable pci passthrough
edit /etc/default/grub
```
GRUB_CMDLINE_LINUX_DEFAULT="quiet intel_iommu=on"
```
edit /etc/modules
```
vfio
vfio_iommu_type1
vfio_pci
vfio_virqfd
```
```
update-grub
reboot
```
## step 3: create pfsense vm
- Create a VM with 1 virtio interface.
- Boot vm and install as EMBEDDED.
- Shutdown VM after first reboot.
- Enable autostart in options of VM.
- Locate your ethernet card using "lspci". The address should be in the form of: 04:00.0.

edit /etc/pve/qemu-server/VMID.conf
```
serial0: socket
hostpci0: 04:00.0
```

## step 4: change hypervisor network
edit /etc/network/interfaces
```
auto lo
iface lo inet loopback

auto vmbr0
iface vmbr0 inet manual
        ovs_type OVSBridge
        ovs_ports int0

allow-vmbr0 int0
iface int0 inet static
        address  10.0.11.2
        netmask  255.255.255.0
        gateway  10.0.11.1
        ovs_type OVSIntPort
        ovs_bridge vmbr0
        ovs_options tag=11
```
edit /etc/hosts
```
127.0.0.1 localhost.localdomain localhost
10.0.11.2 xxx xxx pvelocalhost
```
```
reboot
```
## step 5: configure pfsense
- Back to LARA, use 'qm terminal VMID' to connect to pfsense VM.
- Finish initial setup.
WAN -> em0 -> v4/DHCP4: XXX.XXX.XXX.XXX/XX
LAN -> vtnet0.11 -> v4: 10.0.11.1/24 (vlan 11 in this case)
- Enter shell (option 8), disable pf 'pfctl -d'
- Go to https://XXX.XXX.XXX.XXX and System -> Advanced -> Networking
check 'Disable hardware checksum offload'. (Not sure why, but that hindered me from connecting to webui/ssh on hypervisor.)

## step 6: setup port forwardring or vpn to you liking..
