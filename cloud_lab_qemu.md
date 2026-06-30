# Cloud Lab with QEMU

This document explains how to create a local Cloud Lab environment for testing Ansible playbooks. This guide was written and tested on Ubuntu as the host operating system.

## VM Count and Images

Start by defining the number of virtual machines you want to run.

```bash
export VM_AMOUNT=3
```

Then create an image for each virtual machine. The process for creating the base image is described [here](https://tandiljuan.github.io/en/blog/2024/12/vm-guest-os-debian/).

```bash
for i in $(seq 1 $VM_AMOUNT); do
    leading=$(printf "%02d" $i)
    xz -dk debian-bullseye-amd64.qcow2.xz
    mv debian-bullseye-amd64.qcow2 "debian-bullseye-amd64-${leading}.qcow2"
done
```

## Create Bridge and TAP Devices with a DHCP Server

### Bridge

First, create a bridge device on the host and assign it the IP address `10.0.10.1`. This bridge enables communication between virtual machines and provides a network where a DHCP server can assign IP addresses to them.

```bash
sudo ip link add qemubr0 type bridge
sudo ip link set qemubr0 up
sudo ip addr add 10.0.10.1/24 dev qemubr0
```

### TAP

Now create one TAP device for each virtual machine.

```bash
for i in $(seq 1 $VM_AMOUNT); do
    leading=$(printf "%02d" $i)
    sudo ip tuntap add "tap${leading}" mode tap user $USER
    sudo ip link set "tap${leading}" master qemubr0
    sudo ip link set "tap${leading}" up
done
```

The diagram below shows the network connections between the devices created above.

```
             Host
              |
           qemubr0
         (10.0.10.1)
         /         \
    tap01           tap02
(10.0.10.101)   (10.0.10.102)
      |               |
     VM1             VM2
```

### Configure the Bridge Device for QEMU

Before starting the VMs, ensure that QEMU is allowed to use the bridge device.

```bash
sudo mkdir -p /etc/qemu
echo 'allow qemubr0' | sudo tee /etc/qemu/bridge.conf > /dev/null
```

Also verify that the helper binary has the correct permissions.

```bash
sudo chown root:root /usr/lib/qemu/qemu-bridge-helper
sudo chmod 4755 /usr/lib/qemu/qemu-bridge-helper
```

### DHCP on the Host

Next, configure a simple DHCP server using **dnsmasq** to assign IP addresses to the virtual machines.

The following command creates a dnsmasq configuration file where:

* `interface`: Listens only on the `qemubr0` interface.
* `bind-interfaces`: Restricts dnsmasq to the specified interfaces.
* `dhcp-range`: Defines the pool of IP addresses available to clients.
* `option:router`: Sets `10.0.10.1` as the default gateway.
* `option:dns-server`: Sets `10.0.10.1` as the DNS server.

```bash
sudo tee /etc/dnsmasq.d/qemu.conf > /dev/null <<'EOF'
interface=qemubr0
bind-interfaces

dhcp-range=10.0.10.50,10.0.10.99,255.255.255.0,12h

dhcp-option=option:router,10.0.10.1
dhcp-option=option:dns-server,10.0.10.1

EOF
```

Now append static DHCP reservations to the configuration so that each VM always receives the same hostname and IP address based on its MAC address.

```bash
for i in $(seq 1 $VM_AMOUNT); do
    leading=$(printf "%02d" $i)
    echo "dhcp-host=52:54:00:00:00:${leading},vm${leading},10.0.10.1${leading}" | sudo tee -a /etc/dnsmasq.d/qemu.conf
done
```

Validate the configuration file with the following command.

```bash
dnsmasq --test --conf-file=/etc/dnsmasq.d/qemu.conf
```

To troubleshoot dnsmasq, run it in the foreground and inspect the logs printed to standard output.

```bash
sudo dnsmasq --no-daemon --conf-file=/etc/dnsmasq.d/qemu.conf
```

If the configuration is valid, start dnsmasq in the background. The PID is stored in a temporary file so the process can be stopped later.

```bash
sudo dnsmasq --conf-file=/etc/dnsmasq.d/qemu.conf --pid-file=/tmp/dnsmasq_qemu.pid
```

## Run Virtual Machines

With the network configuration in place, you can start the virtual machines. Each VM exposes its internal SSH port (`22`) through a host port in the format `22XX`, where `XX` is the VM number with a leading zero. For example, the first VM uses port `2201`.

Define the CPU and RAM allocation for each virtual machine. You can also define default values to be used if specific values are not provided for a virtual machine.

```bash
export SMP_DEFAULT=1
export MEM_DEFAULT="1G"
export SMP_LIST=(2)
export MEM_LIST=("2G")
```

Start the virtual machines using the following command:

```bash
for i in $(seq 1 $VM_AMOUNT); do
    leading=$(printf "%02d" $i)
    qemu-system-x86_64  \
        -machine accel=kvm,type=q35 \
        -cpu host \
        -smp "${SMP_LIST[$(( i - 1 ))]:-$SMP_DEFAULT}" \
        -m "${MEM_LIST[$(( i - 1 ))]:-$MEM_DEFAULT}" \
        -device virtio-net-pci,netdev=net0 \
        -netdev "user,id=net0,hostfwd=tcp::22${leading}-:22" \
        -device "virtio-net-pci,netdev=net1,mac=52:54:00:00:00:${leading}" \
        -netdev "tap,id=net1,ifname=tap${leading},script=no,downscript=no" \
        -drive "if=virtio,format=qcow2,file=debian-bullseye-amd64-${leading}.qcow2" \
        -nographic \
        -display none &> /dev/null &
done
```

Connect to a VM with SSH to verify that everything is working correctly.

```bash
ssh -v -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null -p 2201 admin@localhost
```

From inside the VM, verify internet connectivity with the following command.

```bash
sudo lft -p 8.8.8.8
```

You can also test connectivity to another VM, if more than one VM is running.

```bash
ping 10.0.10.102
```

### SSH

To simplify access and avoid typing passwords in this disposable lab environment, create an SSH key using the Ed25519 algorithm.

```bash
ssh-keygen -t ed25519 -C "admin@qemu" -f ~/.ssh/for_qemu_admin
```

Then copy the public key to each virtual machine.

```bash
for i in $(seq 1 $VM_AMOUNT); do
    leading=$(printf "%02d" $i)
    ssh-copy-id -i ~/.ssh/for_qemu_admin.pub \
        -o stricthostkeychecking=no \
        -o userknownhostsfile=/dev/null \
        -p "22${leading}" admin@localhost
done
```

You can also update your SSH client configuration to add an entry for each virtual machine.

```bash
for i in $(seq 1 $VM_AMOUNT); do
    leading=$(printf "%02d" $i)
    cat <<EOF >> ~/.ssh/config

Host vm${leading}
  Hostname localhost
  Port 22${leading}
  User admin
  IdentityFile ~/.ssh/for_qemu_admin
  ServerAliveInterval 60
  ServerAliveCountMax 3
  IdentitiesOnly yes
  StrictHostKeyChecking no
  UserKnownHostsFile /dev/null
  LogLevel QUIET
EOF
done
```

## Cleanup

This section describes how to shut down and remove the lab environment when you are finished.

### Stop the VMs

First, stop all running virtual machines.

```bash
for i in $(seq 1 $VM_AMOUNT); do
    leading=$(printf "%02d" $i)
    ssh "vm${leading}" sudo poweroff
done
```

If you do not plan to use the VMs again, it is safe to remove the image files.

```bash
for i in $(seq 1 $VM_AMOUNT); do
    leading=$(printf "%02d" $i)
    rm "debian-bullseye-amd64-${leading}.qcow2"
done
```

### Stop the DHCP Server

Stop the custom dnsmasq instance.

```bash
# Kill daemon dnsmasq
sudo kill $(cat /tmp/dnsmasq_qemu.pid)
```

### Remove Network Devices

Finally, remove the network devices created for the lab environment.

```bash
for i in $(seq 1 $VM_AMOUNT); do
    leading=$(printf "%02d" $i)
    sudo ip link delete "tap${leading}"
done
sudo ip link delete qemubr0
```
