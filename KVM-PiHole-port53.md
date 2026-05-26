# KVM, Rocky Linux 9, and Pi‑hole Port 53 Conflict

## What I Did
1. Installed KVM on my Ubuntu server (qemu-kvm, libvirt, virt-manager).
2. Created a Rocky Linux 9 virtual machine using virt-install (text‑only, no GUI).
3. Already had Pi‑hole (unbound) in a Docker container listening on port 53 for DNS.

## The Problem
After creating the Rocky VM, Pi‑hole stopped working.  
`sudo ss -tulpn | grep :53` showed dnsmasq listening on 192.168.122.1:53 – the IP of the default KVM virtual network.

## Why It Happened
KVM's libvirt starts a dnsmasq process to provide DNS/DHCP to VMs on the default network (virbr0, IP 192.168.122.1). That dnsmasq grabbed port 53 (UDP/TCP), which Pi‑hole also needed. Two services cannot share the same port – Pi‑hole's DNS resolver failed.

## How I Fixed It
1. Edited the default network configuration:
   sudo virsh net-edit default
2. Inside the <ip> section, added:
   <dns enable='no'/>
3. Restarted the network:
   sudo virsh net-destroy default
   sudo virsh net-start default

## Verification
After the fix, Pi‑hole was restored:
sudo ss -tulpn | grep :53
Now only Pi‑hole (or docker-proxy) shows on port 53.
sudo docker exec pihole-unbound pihole status
Output: [✓] FTL is listening on port 53 and [✓] Pi‑hole blocking is enabled.

## Current Status (as of now)
- Pi‑hole: Working normally, blocking ads and telemetry across my home network.
- Rocky Linux 9 VM: The VM still exists but I have not powered it on since the conflict. It is untested.
- Next step: Eventually power on the Rocky VM to confirm networking works without libvirt's DNS.

## Lesson Learned
- KVM's default network starts dnsmasq on port 53.
- If you already run a DNS server (Pi‑hole, Unbound, BIND), you must either:
  - Disable libvirt's DNS (<dns enable='no'/>), or
  - Change libvirt's DNS port, or
  - Move Pi‑hole to a different port (not recommended for DNS).
- Always check for port conflicts with ss -tulpn when adding new services.
- Document what you actually know, not what you assume.
