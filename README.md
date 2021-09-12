<!-- vim-markdown-toc GFM -->

* [Ansible Role: Virtual Infrastructure](#ansible-role-virtual-infrastructure)
	* [Requirements](#requirements)
		* [KVM host](#kvm-host)
			* [Fedora](#fedora)
			* [CentOS 7](#centos-7)
			* [CentOS 8](#centos-8)
			* [Debian](#debian)
			* [Ubuntu](#ubuntu)
			* [openSUSE](#opensuse)
			* [Using routed networks](#using-routed-networks)
			* [Configuring bridges with NetworkManager](#configuring-bridges-with-networkmanager)
				* [Linux bridge](#linux-bridge)
					* [Using Linux bridge in inventory](#using-linux-bridge-in-inventory)
				* [Open vSwitch (OVS) bridge](#open-vswitch-ovs-bridge)
					* [Using ovs-bridge in inventory](#using-ovs-bridge-in-inventory)
		* [Guest Cloud images](#guest-cloud-images)
	* [Role Variables](#role-variables)
	* [Dependencies](#dependencies)
	* [Example Inventory](#example-inventory)
		* [Multiple KVM hosts](#multiple-kvm-hosts)
	* [Example Playbook](#example-playbook)
		* [Grab the cloud image](#grab-the-cloud-image)
		* [Run the playbook](#run-the-playbook)
		* [Cleanup](#cleanup)
		* [Post setup configuration](#post-setup-configuration)
	* [License](#license)
	* [Author Information](#author-information)

<!-- vim-markdown-toc -->


# Ansible Role: Virtual Infrastructure

This role is designed to define and manage networks and guests on one or more
KVM hosts. Ansible's `--limit` option lets you manage them individually or as
a group.

It is really designed for dev work, where the KVM host is your local machine,
you have `sudo` and talk to `libvirtd` at `qemu:///system` however it also
works on remote hosts.

Setting guest states to _running_, _shutdown_, _destroyed_ or _undefined_ (to
delete and clean up) are supported.

You can set whatever memory, CPU, disks and network cards you want for your
guests, either via hostgroups or individually. A mixture of multiple disks is
supported, including _scsi_, _sata_, _virtio_ and even _nvme_ (on supported
distros).

You can create private NAT libvirt networks on the KVM host and then put VMs on
any number of them. Guests can use those libvirt networks or _existing_ Linux
bridge devices (e.g. `br0`) and Open vSwitch (OVS) bridge on the KVM host (this
won't create bridges on the host, but it will check that the bridge interface
exists). You can specify the model of network card as well as the MAC for each
interface if you require, however it defaults to an idempotent address based on
the hostname.

You can also create routed libvirt networks on the KVM host and then put VMs on
any number of them. In this case, a new bridge is created with the name you
specify (e.g. `br1`), wired to an _existing_ interface (e.g. `eth0`). You can
specify the MAC for each interface if you require.

This supports various distros and uses their qcow2 [cloud
images](#guest-cloud-images) for convenience (although you could use your own
images). I've tested CentOS, Fedora, Debian, Ubuntu and openSUSE.

The qcow2 cloud base images to use for guests are specified as variables in the
inventory and should exist under libvirt images directory (default is
`/var/lib/libvirt/images/`). That is to say, this won't download the images for
you automatically.

Guest qcow2 boot images are created from those base images. By default these
use the cloud image as a backing file, however it also supports cloning
instead. You can create additional disks as you like. You can also choose to
keep any disk image rather than deleting it when a VM is undefined. The
cloud-init ISOs are created automatically and attached to the guest to
configure it on boot.

The timezone will be set to match the KVM host by default and the ansible user
will be used for the guest, along with your public SSH keys on the KVM host
(you can override that). Host entries are added to `/etc/hosts` on the KVM host
and it also modifies ansible user's SSH config and adds the fingerprint to
`known_hosts` so that you can SSH straight in (which it tests as part of the
deploy). You can set a root password if you really want to.

With all that, you could define and manage OpenStack/Swift/Ceph clusters of
different sizes with multiple networks, disks and even distros!

## Requirements

All that's really needed is a Linux host, capable of running KVM, some guest
images and a basic inventory. Ansible will do the rest (on supported distros).

A working x86_64 KVM host where the user running Ansible can communicate with
`libvirtd` via `sudo`.

It expects hardware support for KVM in the CPU so that we an create accelerated
guests and pass the CPU through (supports nested virtualisation).

You may need Ansible and Jinja >= 2.8 because this does things like 'equalto'
comparisons.

I have tested this on CentOS 8, Fedora 3x, Debian 10, Ubuntu Bionic/Eoan and
openSUSE 15 hosts, but other Linux machines probably work.

At least one SSH key pair on your KVM host (the Ansible will generate one if
missing). The SSH key used for guests should not require an SSH passphrase, if
on a remote KVM host. If local, then make sure you've added it to ssh agent.

Several user space tools are also required on the KVM host (the Ansible will
install these on supported hosts).

* qemu-img
* osinfo-query
* virsh
* virt-customize
* virt-sysprep

Download the guest images you want to use ([this is what I
downloaded](#guest-cloud-images)) and put them in libvirt images path (usually
`/var/lib/libvirt/images/`). This will check that the images you specified
exist and error if they are not found.

### KVM host

Here are some instructions for configuring your KVM host, in case they are
useful.

**NOTE:** This role will do all this for you including install KVM, libvirtd
and other required packages on supported distros and also make sure the
libvirtd is running.


#### Fedora

```bash
# Create SSH key if you don't have one (don't set a passphrase if remote KVM host)
ssh-keygen

# libvirtd
sudo dnf install -y @virtualization
sudo systemctl enable --now libvirtd

# Ansible
sudo dnf install -y ansible

# Other deps (installed by playbook)
sudo dnf install -y \
git \
genisoimage \
libguestfs-tools-c \
libosinfo \
python3-libvirt \
python3-lxml \
qemu-img \
virt-install
```

#### CentOS 7

CentOS 7 won't work until we have `libselinux-python3` package, which is coming in 7.8...

* https://bugzilla.redhat.com/show_bug.cgi?id=1719978
* https://bugzilla.redhat.com/show_bug.cgi?id=1756015

But here are (hopefully) the rest of the steps for when it is available.

```bash
# Create SSH key if you don't have one
ssh-keygen

# libvirtd
sudo yum groupinstall -y "Virtualization Host"
sudo systemctl enable --now libvirtd

# Ansible and other deps
sudo yum install -y epel-release
sudo yum install -y python36
pip3 install --user ansible

sudo yum install -y \
git \
genisoimage \
libguestfs-tools-c \
libosinfo \
python36-libvirt \
python36-lxml \
libselinux-python3 \
qemu-img \
virt-install
```

#### CentOS 8

```bash
# Create SSH key if you don't have one
ssh-keygen

# libvirtd
sudo dnf groupinstall -y "Virtualization Host"
sudo systemctl enable --now libvirtd

# Ansible
sudo dnf install -y epel-release
sudo dnf install -y ansible

# Other deps (installed by playbook)
sudo dnf install -y \
git \
genisoimage \
libguestfs-tools-c \
libosinfo \
python3 \
python3-libvirt \
python3-lxml \
qemu-img \
virt-install
```

#### Debian

```bash
# Create SSH key if you don't have one
ssh-keygen

# libvirtd
sudo apt update
sudo apt install -y --no-install-recommends qemu-kvm libvirt-clients libvirt-daemon-system
sudo systemctl enable --now libvirtd

# Ansible
sudo apt install -y gnupg2
echo 'deb http://ppa.launchpad.net/ansible/ansible/ubuntu trusty main' | sudo tee -a /etc/apt/sources.list
sudo apt-key adv --keyserver keyserver.ubuntu.com --recv-keys 93C4A3FD7BB9C367
sudo apt update
sudo apt install -y ansible

# Other deps (installed by playbook)
sudo apt install -y --no-install-recommends \
cloud-image-utils \
dnsmasq \
git \
genisoimage \
libguestfs-tools \
libosinfo-bin \
python3-libvirt \
python3-lxml \
qemu-utils \
virtinst
```

#### Ubuntu

```bash
# Create SSH key if you don't have one
ssh-keygen

# libvirtd
sudo apt update
sudo apt install -y --no-install-recommends libvirt-clients libvirt-daemon-system qemu-kvm
sudo systemctl enable --now libvirtd

# Ansible
sudo apt install -y software-properties-common
sudo apt-add-repository --yes --update ppa:ansible/ansible
sudo apt install -y ansible

# Other deps (installed by playbook)
sudo apt install -y --no-install-recommends \
dnsmasq \
git \
genisoimage \
libguestfs-tools \
libosinfo-bin \
python3-libvirt \
python3-lxml \
qemu-utils \
virtinst
```

#### openSUSE

If you're running JeOS, we need to change the kernel to `kernel-default` as
`kernel-default-base` which comes with JeOS is missing KVM modules.

```bash
# Create SSH key if you don't have one
ssh-keygen

# Install suitable kernel
sudo zypper install kernel-default
sudo reboot
```

Continue after reboot.

```bash
# libvirtd
sudo zypper install -yt pattern kvm_server kvm_tools
sudo systemctl enable --now libvirtd

# Ansible
sudo zypper install -y ansible

# Other deps (installed by playbook)
sudo zypper install -y \
git \
guestfs-tools \
libosinfo \
mkisofs \
python3-libvirt-python \
python3-lxml \
qemu-tools \
virt-install
```

#### Using routed networks

You can route traffic into a newly created bridge by specifying forward *type: route*.
This code supports automatic creation of a new bridge named `bridge_dev` which will be
wired onto an existing interface in the host, specified by parameter `host_dev`.

The example below shows how a bridge can be created, supporting both IPv4 and IPv6:

```yaml
kvmhost:
  hosts:
    localhost:
      ansible_connection: local
      ansible_python_interpreter: /usr/bin/python3
      virt_infra_host_libvirt_url: qemu:///system
  vars:
    virt_infra_host_networks:
      present:
        - name: example
          domain: f901.example.com
          type: route
          host_dev: eth0
          bridge_dev: virbr1
          bridge_stp: on
          bridge_delay: 0
          mac: 52:54:00:f9:01:00
          ip_address: 10.249.1.1
          ip_netmask: 255.255.255.0
          dhcp_start: 10.249.1.11
          dhcp_end: 10.249.1.254
          ip6_address: 2001:0db8::f901:1
          ip6_prefix: 64
          dhcp6_start: 2001:0db8::f901:0000
          dhcp6_end: 2001:0db8::f901:00ff
```

Notes:

1. The IPv6 block 2001:0db8/32 as shown above is provided for the sake of documentation
   purposes only. You will have to substitute that by your own delegated /48 block
   (in general) given to you by your IPv6 provider or by a IPv6 over IPv4 tunnelling
   solution such as [Hurricane Electric's tunnel broker service](http://tunnelbroker.net/).

2. It's highly recommended that you stick with `ip6_prefix: 64`, since it is the
   recommended setting in libvirt documentation.

#### Configuring bridges with NetworkManager

This code supports connecting VMs to both Linux and Open vSwitch bridges, but
they must already exist on the KVM host.

Here is how to convert an existing ethernet device into a bridge. Be careful if
doing this on a remote machine with only one connection! Make sure you have
some other way to log in (e.g. console), or maybe add additional interfaces
instead.

First, export the the device you want to convert so we can easily reference it
later (e.g.  `eth1`).

```bash
export NET_DEV="eth1"
```

Now list the current NetworkManager connections for your device exported above
so we know what to disable later.

```bash
sudo nmcli con |egrep -w "${NET_DEV}"
```

This might be something like `System eth1` or `Wired connection 1`, let's export
it too for later reference.

```bash
export NM_NAME="Wired connection 1"
```

##### Linux bridge

Here is an example of creating a persistent Linux bridge with NetworkManager.
It will take a device such as `eth1` (substitute as appropriate) and convert it
into a bridge.

Remember your device's existing NetworkManager connection name from above, you
will use it below (e.g. `Wired connection 1`).

```bash
export NET_DEV=eth1
export NM_NAME="Wired connection 1"
sudo nmcli con add ifname br0 type bridge con-name br0
sudo nmcli con add type bridge-slave ifname "${NET_DEV}" master br0
```

OK now you have your bridge device! Note the bridge will have a different MAC
address to the underlying device, so if you're expecting it to get a specific
address, you'll need to update your DHCP static lease.

```bash
sudo ip link show dev br0
```

Disable the current NetworkManager config for the device so that it doesn't
conflict with the bridge (don't delete it yet, you may lose connection if
you're using it for SSH).

```
sudo nmcli con modify id "${NM_NAME}" ipv4.method disabled ipv6.method disabled
```

Now you can either simply `reboot`, or stop the current interface and bring up
the bridge in one command. Remember that the bridge will have a new MAC address
so it will get a new IP, unless you've updated your DHCP static leases!

```bash
sudo nmcli con down "${NM_NAME}" ; sudo nmcli con up br0
```

As mentioned above, by default the Linux bridge will get an address via DHCP.
If you don't want it to be on the network (you might have another dedicated
interface) then disable DHCP on it.

```bash
sudo nmcli con modify id br0 ipv4.method disabled ipv6.method disabled
```

###### Using Linux bridge in inventory

There's nothing to do on the `kvmhost` side of the inventory.

For any guests you want to connect to the bridge, simply specify it in their
inventory. Use `br0` as the `name` of a network under `virt_infra_networks`
with type `bridge`.

```yaml
      virt_infra_networks:
        - name: br0
          type: bridge
```

##### Open vSwitch (OVS) bridge

Here is an example of creating a persistent OVS bridge with NetworkManager. It
will take a device such as `eth1` (substitute as appropriate) and convert it
into an ovs-bridge.

You will need openvswitch installed as well as the OVS NetworkManager plugin
(substitute for your distro).

```bash
sudo dnf install -y NetworkManager-ovs openvswitch
sudo systemctl enable --now openvswitch
sudo systemctl restart NetworkManager
```

Now we can create the OVS bridge (assumes your device is `eth1` and existing
NetworkManager config is `Wired connection 1`, substitute as appropriate).

```bash
export NET_DEV=eth1
export NM_NAME="Wired connection 1"
sudo nmcli con add type ovs-bridge conn.interface ovs-bridge con-name ovs-bridge
sudo nmcli con add type ovs-port conn.interface port-ovs-bridge master ovs-bridge
sudo nmcli con add type ovs-interface slave-type ovs-port conn.interface ovs-bridge master port-ovs-bridge
sudo nmcli con add type ovs-port conn.interface ovs-port-eth master ovs-bridge con-name ovs-port-eth
sudo nmcli con add type ethernet conn.interface "${NET_DEV}" master ovs-port-eth con-name ovs-int-eth
```

Disable the current NetworkManager config for the device so that it doesn't
conflict with the bridge (don't delete it yet, you may lose connection if
you're using it for SSH).

```
sudo nmcli con modify id "${NM_NAME}" ipv4.method disabled ipv6.method disabled
```

Now you can either simply `reboot`, or stop the current interface and bring up
the bridge in one command.

```bash
sudo nmcli con down "${NM_NAME}" ; sudo nmcli con up ovs-slave-ovs-bridge
```

By default the OVS bridge will get an address via DHCP. If you don't want it to
be on the network (you might have another dedicated interface) then disable
DHCP on it.

```bash
sudo nmcli con modify id ovs-slave-ovs-bridge ipv4.method disabled ipv6.method disabled
```

Show the switch config and bridge with OVS tools.

```bash
sudo ovs-vsctl show
```

###### Using ovs-bridge in inventory

Now you can use `ovs-bridge` as the `device` of an `ovs` bridge in your
`kvmhost` inventory `virt_infra_host_networks` entry and it will create the OVS
libvirt networks for you. You can set up multiple VLANs and set one as default
native (if required).

VLAN ranges are supported by defining it as a list.

```yaml
      virt_infra_host_networks:
        present:
          - name: ovs-bridge
            bridge_dev: ovs-bridge
            type: ovs
            portgroup:
              # This is a portgroup with multiple VLANs
              # It is native VLAN 1 and also allows traffic tagged with VLAN 99
              - name: ovs-trunk
                trunk: true
                native_vlan: 1
                vlan:
                  - 1
                  - [20,29]
                  - 99
              # This is portgroup just for native VLAN 1
              - name: default
                native_vlan: 1
                vlan:
                  - 1
              # This is portgroup just for native VLAN 99
              - name: other
                native_vlan: 99
                vlan:
                  - 99
```

You can then specify your VMs to be on specific portgroups and libvirt will
automatically set up the ports for your VMs and you.

```yaml
      virt_infra_networks:
        - name: ovs-bridge
          portgroup: default
          type: ovs
```

Once your VMs are running, you can see their OVS ports with `sudo ovs-vsctl
show`.

### Guest Cloud images

This is designed to use standard cloud images provided by various distros
(OpenStack [provides some
suggestions](https://docs.openstack.org/image-guide/obtain-images.html)).

Make sure the Image you're specifying for your guests already exists under your
libvirt storage dir (by default this is _/var/lib/libvirt/images/_).

I have tested the following guests successfully:

* CentOS 7
  * https://cloud.centos.org/centos/7/images/CentOS-7-x86_64-GenericCloud.qcow2
* CentOS 8
  * https://cloud.centos.org/centos/8/x86_64/images/CentOS-8-GenericCloud-8.1.1911-20200113.3.x86_64.qcow2
* Fedora 33
  * https://download.fedoraproject.org/pub/fedora/linux/releases/33/Cloud/x86_64/images/Fedora-Cloud-Base-33-1.2.x86_64.qcow2
* Fedora 34
  * https://download.fedoraproject.org/pub/fedora/linux/releases/34/Cloud/x86_64/images/Fedora-Cloud-Base-34-1.2.x86_64.qcow2
* Debian 10
  * http://cdimage.debian.org/cdimage/openstack/current-10/debian-10-openstack-amd64.qcow2
* Ubuntu 18.04 LTS
  * http://cloud-images.ubuntu.com/bionic/current/bionic-server-cloudimg-amd64.img
* Ubuntu 20.04 LTS
  * http://cloud-images.ubuntu.com/focal/current/focal-server-cloudimg-amd64.img
* openSUSE 15.3 JeOS
  * http://download.opensuse.org/distribution/leap/15.3/appliances/openSUSE-Leap-15.3-JeOS.x86_64-15.3-OpenStack-Cloud-Current.qcow2

So that we can configure the guest and get its IP, both `cloud-init` and
`qemu-guest-agent` will be installed into you guest's image, just in case.

This can be changed or overridden using the `virt_infra_guest_deps` variable,
which is a list.

Sysprep is also run on the guest image to make sure it's clean of things like
old MAC addresses.

## Role Variables

The role defaults are set in the _defaults/main.yml_ file.

These can be overridden at host or hostgroup level, as necessary. However, the
role is designed to pretty much work out of the box (so long as you have the
default CentOS image).

```yaml
---
# Defaults for virt-infra Ansible role
# Values which are commented out are optional

## Guest related

# Valid guest states are: running, shutdown, destroyed or undefined
virt_infra_state: "running"

# Guests are not autostarted on boot
virt_infra_autostart: "no"

# Guest user, by default this will be set to the same user as KVM host user
virt_infra_user: "{{ hostvars[kvmhost].ansible_env.USER }}"

# Password of default user (consider a vault if you need secure passwords)
# No root password by default
virt_infra_password: "password"
#virt_infra_root_password:

# VM specs for guests
# See virt-install manpage for supported values
virt_infra_ram: "1024"
virt_infra_ram_max: "{{ virt_infra_ram }}"
virt_infra_cpus: "1"
virt_infra_cpus_max: "{{ virt_infra_cpus }}"
virt_infra_cpu_model: "host-passthrough"
virt_infra_machine_type: "q35"

# SSH keys are a list, you can add more than one
# If not specified, we default to all public keys on KVM host
virt_infra_ssh_keys: []

# If no SSH keys are specified or found on the KVM host, we create one with this
virt_infra_ssh_key_size: "2048"
virt_infra_ssh_key_type: "rsa"

# Whether to enable SSH password auth
virt_infra_ssh_pwauth: true

# Whether to use cloud-init to configure networking on guest
virt_infra_network_config: false

# Networks are a list, you can add more than one
# "type" is optional, both "nat" and "bridge" are supported
#  - "nat" is default type and should be a libvirt network
#  - "bridge" type requires the bridge interface as the name (e.g. name: "br0") which also must already be setup on KVM host
# "model" is also optional
virt_infra_networks:
  - name: "default"
    type: "nat"
    model: "virtio"

# Disks, support various libvirt options
# We generally don't set them though and leave it to hypervisor default
# See virt-install manpage for supported values
virt_infra_disk_size: "20"
virt_infra_disk_bus: "scsi"
virt_infra_disk_io: "threads"
virt_infra_disk_cache: "writeback"

# Disks are a list, you can add more than one
# If you override this, you must still include 'boot' device first in the list
# Only 'name' is required, others are optional (default size is 20GB)
# All guests require at least a boot drive (which is the default)
virt_infra_disks:
  - name: "boot"
    size: "{{ virt_infra_disk_size }}"
    bus: "{{ virt_infra_disk_bus }}"
#   io: "{{ virt_infra_disk_io }}"
#   cache: "{{ virt_infra_disk_cache }}"

# Default distro is CentOS 8, override in guests or groups
virt_infra_distro_image: "CentOS-Stream-GenericCloud-8-20210603.0.x86_64.qcow2"

# Determine supported variants on your KVM host with command, "osinfo-query os"
# This doesn't really make much difference to the guest, maybe slightly different bus
# You could probably just set this as "centos7.0" for all distros, if you wanted to
#virt_infra_variant: "centos7.0"

# These distro vars are here for reference and convenience
virt_infra_distro: "centos"
virt_infra_distro_release: "7"
virt_infra_distro_image_url: "https://cloud.centos.org/centos/7/images/CentOS-7-x86_64-GenericCloud.qcow2"
virt_infra_distro_image_checksum_url: "https://cloud.centos.org/centos/7/images/sha256sum.txt"

## KVM host related

# Connect to system libvirt instance
virt_infra_host_libvirt_url: "qemu:///system"

# Path where disk images are kept
virt_infra_host_image_path: "/var/lib/libvirt/images"

# Disable qemu security driver by default
# This is overridden in distro specific vars
virt_infra_security_driver: "none"

# Virtual BMC is disabled by default
virt_infra_vbmc: false

# By default we install with pip, but if you prefer to do it manually, set this to false
virt_infra_vbmc_pip: true

# Default vbmc service, override if something else on your distro
virt_infra_vbmc_service: vbmcd

# Networks on kvmhost are a list, you can add more than one
# You can create and remove NAT networks on kvmhost (creating bridges not supported)
# The 'default' network is the standard one shipped with libvirt
# By default we don't remove any networks (empty absent list)
virt_infra_host_networks:
  absent: []
  present:
    - name: "default"
      type: "nat"
      ip_address: "192.168.122.1"
      subnet: "255.255.255.0"
      dhcp_start: "192.168.122.2"
      dhcp_end: "192.168.122.254"

# Command for creating ISO images
virt_infra_mkiso_cmd: genisoimage

# List of binaries to check for on KVM Host
virt_infra_host_deps:
  - qemu-img
  - osinfo-query
  - virsh
  - virt-customize
  - virt-sysprep

# Comma separated list of packages to install into guest disks
virt_infra_guest_deps:
  - cloud-init
  - qemu-guest-agent
```

## Dependencies

None

## Example Inventory

The separate [virt-infra repo](https://github.com/csmart/virt-infra-ansible)
has sample inventory files and site playbook to call the role, which might be
helpful.

It might be best if the inventories are split into multiple files for ease of
management, under an _inventory_ directory or such. I suggest a common
_kvmhost.yml_ then separate inventory files for each group of guests, e.g.
_openstack.yml_. When running ansible, you include the whole directory as the
inventory source.

The inventory used with this role must include a hostgroup called _kvmhost_ and
other hostgroups for guests.

Custom settings can be provided for each host or group of hosts in the
inventory.

To create a new group of guests to manage, create a new yml file under the
inventory directory. For example, if you wanted a set of guests for OpenStack,
you could create an `openstack.yml` file and populate it as required.

To manage specific hosts or groups, simply use Ansible's `--limit` option to
specify the hosts or hostgroups (must also include _kvmhost_ group). This way
you can use the one inventory for lots of different guests and manage them
separately.

The KVM host is where the libvirt networks are created and therefore specified
as vars under that hostgroup.

Here is an example inventory file _kvmhost.yml_ in YAML format. It is
specifying _kvmhost_ as localhost with a local connection. Note there are two
networks being created (_default_ and _example_) and one which is removed
(_other_).


```yaml
---
## YAML based inventory, see:
## https://docs.ansible.com/ansible/latest/plugins/inventory/yaml.html
#
kvmhost:
  hosts:
    # Put your KVM host connection and settings here
    localhost:
      ansible_connection: local
      ansible_python_interpreter: /usr/bin/python3
  vars:
    # Networks are a list, you can add more than one
    # You can create and remove NAT networks on kvmhost (creating bridges not supported)
    # The 'default' network is the standard one shipped with libvirt
    # By default we don't remove any networks (empty absent list)
    virt_infra_host_networks:
      absent:
        - name: "other"
      present:
        - name: "default"
          ip_address: "192.168.112.1"
          subnet: "255.255.255.0"
          dhcp_start: "192.168.112.2"
          dhcp_end: "192.168.112.254"
        - name: "example"
          ip_address: "192.168.113.1"
          subnet: "255.255.255.0"
          dhcp_start: "192.168.113.2"
          dhcp_end: "192.168.113.99"
```

Here's an example guest inventory called _simple.yml_ which defines CentOS 8
guests in a group called _simple_, using the role defaults.

```
---
## YAML based inventory, see:
## https://docs.ansible.com/ansible/latest/plugins/inventory/yaml.html
#
simple:
  hosts:
    centos-simple-[0:2]:
      ansible_python_interpreter: /usr/libexec/platform-python
```

If you want a group of VMs to all be the same, set the vars at the hostgroup
level. You can still override hostgroup vars with individual vars for specific
hosts, if required.

Here's an example setting various hostgroup and individual host vars.

```yaml
---
## YAML based inventory, see:
## https://docs.ansible.com/ansible/latest/plugins/inventory/yaml.html
#
example:
  hosts:
    centos-7-example:
      virt_infra_state: shutdown
      virt_infra_timezone: "Australia/Melbourne"
      ansible_python_interpreter: /usr/bin/python
      virt_infra_networks:
        - name: "br0"
          type: bridge
        - name: "extra_network"
          type: nat
          model: e1000
        - name: ovs-bridge
          portgroup: ovs-portgroup
          type: ovs
      virt_infra_disks:
        - name: "boot"
        - name: "nvme"
          size: "100"
          bus: "nvme"
    centos-8-example:
      virt_infra_timezone: "Australia/Adelaide"
      ansible_python_interpreter: /usr/libexec/platform-python
    opensuse-15-example:
      virt_infra_distro: opensuse
      virt_infra_distro_image: openSUSE-Leap-15.1-JeOS.x86_64-15.1.0-OpenStack-Cloud-Current.qcow2
      virt_infra_variant: opensuse15.1
      virt_infra_disks:
        - name: "boot"
          bus: "scsi"
    ubuntu-eoan-example:
      virt_infra_cpu: 2
      virt_infra_distro: ubuntu
      virt_infra_distro_image: eoan-server-cloudimg-amd64.img
      virt_infra_variant: ubuntu18.04
  vars:
    virt_infra_ram: 1024
    virt_infra_disks:
      - name: "boot"
      - name: "data"
        bus: "sata"
    virt_infra_networks:
      - name: "example"
        type: nat
```

### Multiple KVM hosts

You can also specify multiple KVM hosts.

```yaml
---
kvmhost:
  hosts:
    kvmhost1:
    kvmhost2:
    kvmhost3:
  vars:
    ansible_python_interpreter: /usr/bin/python3
    virt_infra_host_networks:
      absent: []
      present:
        - name: "default"
          ip_address: "192.168.112.1"
          subnet: "255.255.255.0"
          dhcp_start: "192.168.112.2"
          dhcp_end: "192.168.112.254"

```

To have a VM land on a specific KVM host, you must add the variable `kvmhost`
with a string that matches a KVM host from the `kvmhost` group.

For example, six CentOS hosts across three KVM hosts:

```yaml
---
simple:
  hosts:
    simple-centos-[1:2]:
      kvmhost: kvmhost1
    simple-centos-[3:4]:
      kvmhost: kvmhost2
    simple-centos-[5:6]:
      kvmhost: kvmhost3
  vars:
    ansible_python_interpreter: /usr/libexec/platform-python
    virt_infra_distro_image: "CentOS-Stream-GenericCloud-8-20210603.0.x86_64.qcow2"
```

If no kvmhost is specified for a VM it will default to the first KVM host in
the `kvmhost` group (i.e. kvmhost[0]) which matches the original behaviour for
the role.

Validation checks have been updated to make sure that all of the KVM hosts are
valid and that any specified KVM host for a VM is in the `kvmhost` group.

To group VMs on certain KVM hosts, consider making child groups and specify
kvmhost at the child group level.

For example, those CentOS guests again:

```yaml
---
simple:
  hosts:
    simple-centos-[1:6]:
  vars:
    ansible_python_interpreter: /usr/libexec/platform-python
    virt_infra_distro_image: "CentOS-Stream-GenericCloud-8-20210603.0.x86_64.qcow2"
  children:
    simple_kvmhost1:
      hosts:
        simple-centos-[1:2]:
      vars:
        kvmhost: kvmhost1
    simple_kvmhost2:
      hosts:
        simple-centos-[3:4]:
      vars:
        kvmhost: kvmhost2
    simple_kvmhost3:
      hosts:
        simple-centos-[5:6]:
      vars:
        kvmhost: kvmhost3
```


## Example Playbook

I've tried to keep the Ansible as a simple, logical set of steps and not get
too tricky. Having said that, the playbook is quite specific.

There are some tasks which can only be run on the KVM host and others in a
specific order.

This code will help by running a bunch of validation checks on the KVM host and
for your guest configs to try to catch anything that's not right.

Here is an example playbook _virt-infra.yml_ which calls the role.

```yaml
---
- hosts: all
  gather_facts: no
  roles:
    - ansible-role-virt-infra
```

### Grab the cloud image

Before we run the playbook, download the CentOS cloud image.

```bash
curl -O https://cloud.centos.org/centos/8/x86_64/images/CentOS-Stream-GenericCloud-8-20210603.0.x86_64.qcow2
sudo mv -iv CentOS-Stream-GenericCloud-8-20210603.0.x86_64.qcow2 /var/lib/libvirt/images/
```

### Run the playbook

Now run the Ansible playbook against the kvmhost and the guests in the _simple_
group using the inventory above (note that we limit to both the _kvmhost_ and
the _simple_ hostgroups).

```bash
ansible-playbook \
--ask-become-pass \
--inventory ./inventory.d \
--limit kvmhost,simple \
./virt-infra.yml
```

You can also override a number of guest settings on the command line.

```bash
ansible-playbook \
--ask-become-pass \
--limit kvmhost,simple \
./virt-infra.yml \
-e virt_infra_root_password=password \
-e virt_infra_disk_size=100 \
-e virt_infra_ram=4096 \
-e virt_infra_ram_max=8192 \
-e virt_infra_cpus=8 \
-e virt_infra_cpus_max=16 \
-e '{ "virt_infra_networks": [{ "name": "br0", "type": "bridge" }] }' \
-e virt_infra_state=running
```

### Cleanup

To delete the guests in a hostgroup you could specify them (or a specific host)
with `--limit` and pass in _virt_infra_state=undefined_ as a command line extra
arg.

This will override the guest state to undefined and if they exist, they will be
deleted.

For example, to delete all VMs in the _simple_ hostgroup.

```bash
ansible-playbook \
--ask-become-pass \
--inventory simple.yml \
--limit kvmhost,simple \
--extra-vars virt_infra_state=undefined \
./virt-infra.yml
```

If you want to just shut them down, try _virt_infra_state=shutdown_
instead.

For example, to shutdown just the _simple-centos-2_ host.

```bash
ansible-playbook \
--ask-become-pass \
--inventory simple.yml \
--limit kvmhost,simple-centos-2 \
--extra-vars virt_infra_state=shutdown \
./virt-infra.yml
```

### Post setup configuration

Once you have set up your infra, you could run another playbook against your
same inventory to do whatever you wanted with those machines...

```yaml
---
- hosts: all,!kvmhost
  tasks:
    - name: Upgrade all packages
      package:
        name: '*'
        state: latest
      become: true
      register: result_package_update
      retries: 30
      delay: 10
      until: result_package_update is succeeded

    - name: Install packages
      package:
        name:
          - git
          - tmux
          - vim
        state: present
      become: true
      register: result_package_install
      retries: 30
      delay: 10
      until: result_package_install is succeeded
```

## License

GPLv3

## Author Information

Chris Smart https://blog.christophersmart.com
