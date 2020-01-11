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
		* [Guest Cloud images](#guest-cloud-images)
	* [Role Variables](#role-variables)
	* [Dependencies](#dependencies)
	* [Example Inventory](#example-inventory)
	* [Example Playbook](#example-playbook)
		* [Grab the cloud image](#grab-the-cloud-image)
		* [Run the playbook](#run-the-playbook)
		* [Cleanup](#cleanup)
		* [Post setup configuration](#post-setup-configuration)
	* [License](#license)
	* [Author Information](#author-information)

<!-- vim-markdown-toc -->


# Ansible Role: Virtual Infrastructure

This role is designed to define and manage networks and guests on a KVM host.
Ansible's _--limit_ option lets you manage them individually or as a group.

It is really designed for dev work, where the KVM host is your local machine,
you have sudo and talk to libvirtd at qemu:///system (although in theory it
supports a remote KVM host).

Setting guest states to _running_, _shutdown_, _destroyed_ or _undefined_ (to
delete and clean up) are supported.

You can set whatever memory, CPU, disks and network cards you want for your
guests, either via hostgroups or individually. A mixture of multiple disks is
supported, including _scsi_, _sata_, _virtio_ and even _nvme_.

You can create private NAT libvirt networks on the KVM host and then put VMs on
any number of them. Guests can use those libvirt networks or _existing_ bridge
devices (e.g. br0) on the KVM host (this won't create bridges on the host, but
it will check that the bridge interface exists).

This supports various distros and uses their qcow2 [cloud
images](#guest-cloud-images) for convenience (although you could use your own
images). I've tested CentOS, Fedora, Debian, Ubuntu and openSUSE.

The qcow2 cloud base images to use for guests are specified as variables in the
inventory and should exist under libvirt images directory (default is
_/var/lib/libvirt/images/_). That is to say, this won't download the images for
you automatically.

Guest qcow2 boot images are created from those base images and cloud-init is
used to configure guests on boot up. The cloud-init ISOs are created
automatically and attached to the guest. Timezone will be set to match the KVM
host by default.

By default, your shell username will also be used for the guest, along with
your public SSH keys on the KVM host (you can override that). Host entries are
added to /etc/hosts on the KVM host so you can SSH straight in (but it doesn't
modify your SSH config yet). You can set a root password if you really want to.

With all that, you could define and manage OpenStack/Swift/Ceph clusters of
different sizes with multiple networks, disks and even distros!

## Requirements

All that's really needed is a Linux host, capable of running KVM, some guest
images and a basic inventory. Ansible will do the rest (on supported distros).

**NOTE:** The Ansible will install KVM, libvirtd and other required packages on
supported distros and also make sure the libvirtd is running.

A working x86_64 KVM host where the user running Ansible can communicate with
libvirtd via sudo.

It expects hardware support for KVM in the CPU so that we an create accelerated
guests and pass the CPU through (supports nested virtualisation).

You may need Ansible and Jinja >= 2.8 because this does things like 'equalto'
comparisons.

I have tested this on CentOS 8, Fedora 3x, Debian 10, Ubuntu Bionic/Eoan and
openSUSE 15 hosts, but other Linux machines probably work.

At least one SSH key pair on your KVM host (the Ansible will generate one if
missing).

Several user space tools are also required on the KVM host (the Ansible will
install these on supported hosts).

* qemu-img
* osinfo-query
* virsh
* virt-customize
* virt-sysprep

Download the guest images you want to use ([this is what I
downloaded](#guest-cloud-images)) and put them in libvirt images path (usually
_/var/lib/libvirt/images/_). This will check that the images you specified
exist and error if they are not found.

### KVM host

Here are some instructions for configuring your KVM host, in case they are
useful.

#### Fedora

```bash
# Create SSH key if you don't have one
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
* Fedora 30
  * https://download.fedoraproject.org/pub/fedora/linux/releases/30/Cloud/x86_64/images/Fedora-Cloud-Base-30-1.2.x86_64.qcow2
* Fedora 31
  * https://download.fedoraproject.org/pub/fedora/linux/releases/31/Cloud/x86_64/images/Fedora-Cloud-Base-31-1.9.x86_64.qcow2
* Debian 10
  * http://cdimage.debian.org/cdimage/openstack/current/debian-10.2.0-openstack-amd64.qcow2
* Ubuntu 16.04 LTS
  * http://cloud-images.ubuntu.com/bionic/current/bionic-server-cloudimg-amd64.img
* Ubuntu 19.10
  * http://cloud-images.ubuntu.com/eoan/current/eoan-server-cloudimg-amd64.img
* openSUSE 15.1 JeOS
  * https://download.opensuse.org/distribution/leap/15.1/jeos/openSUSE-Leap-15.1-JeOS.x86_64-15.1.0-OpenStack-Cloud-Current.qcow2

So that we can configure the guest and get its IP, both cloud-init and
qemu-guest-agent will be installed into you guest's image, just in case.

Sysprep is also run on the guest image to make sure it's clean of things like
old MAC addresses.

## Role Variables

The defaults are set in the _defaults/main.yml_ file.

These can be overridden at host or hostgroup level, as necessary. However, the
role is designed to pretty much work out of the box (so long as you have the
default CentOS image).

```yaml
## Guest related
# Valid guest states are: running, shutdown, destroyed or undefined
virt_infra_state: running

# Guests are not autostarted on boot
virt_infra_autostart: "no"

# Guest user set to match KVM host user
virt_infra_user: "{{ lookup('env', 'USER' )}}"

# Password of default user (consider a vault if you need secure passwords)
# No root password by default
virt_infra_password: "password"
virt_infra_root_password:

# VM specs for guests
virt_infra_ram: "1024"
virt_infra_ram_max: "{{ virt_infra_ram }}"
virt_infra_cpus: "1"
virt_infra_cpus_max: "{{ virt_infra_cpus }}"
virt_infra_cpu_model: "host-passthrough"
virt_infra_machine_type: "q35"

# SSH keys are a list, you can add more than one
# If not specified, we default to all public keys on KVM host
virt_infra_ssh_keys: []

# Whether to enable SSH password auth
virt_infra_ssh_pwauth: true

# Networks for guests are a list, you can add more than one
# "type" is optional, both "nat" and "bridge" are supported
#  - "nat" is default type and should be a libvirt network
#  - "bridge" type requires the bridge interface (e.g. br0) to already exist on KVM host
# "model" is also optional
virt_infra_networks:
  - name: "default"
    type: "nat"
    model: "virtio"

# Disk defaults, support various libvirt options
# We generally don't set them though, and leave it to hypervisor default
virt_infra_disk_size: "20"
virt_infra_disk_bus: "scsi"
virt_infra_disk_io: "threads"
virt_infra_disk_cache: "writeback"

# Disks for guests are a list, you can add more than one
# If you override this, you must still include 'boot' device first in the list
# Only 'name' is required, others are optional (default size is 20GB)
# All guests require at least a boot drive (which is the default)
virt_infra_disks:
  - name: "boot"
    size: "{{ virt_infra_disk_size }}"
    bus: "{{ virt_infra_disk_bus }}"
    io: "{{ virt_infra_disk_io }}"
    cache: "{{ virt_infra_disk_cache }}"

# Default distro is CentOS 7, override in guests or groups
virt_infra_distro_image: "CentOS-7-x86_64-GenericCloud.qcow2"

# Determine supported variants on your KVM host with command, "osinfo-query os"
# This doesn't really make much difference to the guest, maybe slightly different bus
# You could probably just leave this as "centos7.0" for all distros, if you wanted to
virt_infra_variant: "centos7.0"

## KVM host
# Connect to system libvirt instance
virt_infra_host_libvirt_url: "qemu:///system"

# Path where disk images are kept
virt_infra_host_image_path: "/var/lib/libvirt/images"

# Networks on kvmhost are a list, you can add more than one
# You can create and remove NAT networks on kvmhost (creating bridges not supported)
# The 'default' network is the standard one shipped with libvirt
# By default we don't remove any networks (empty absent list)
virt_infra_host_networks:
  absent: []
  present:
    - name: "default"
      ip_address: "192.168.112.1"
      subnet: "255.255.255.0"
      dhcp_start: "192.168.112.2"
      dhcp_end: "192.168.112.254"

# List of binaries to check for on kvmhost
virt_infra_host_deps:
  - qemu-img
  - osinfo-query
  - virsh
  - virt-customize
  - virt-sysprep
```

## Dependencies

None

## Example Inventory

The separate [virt-infra repo](https://github.com/csmart/virt-infra-ansible)
has sample inventory files and site playbook to call the role, which might be
helpful.

It might be best if the inventories are split into multiple files for ease of
management, under an _inventory_ directory or such.

The inventory used with this role must include a hostgroup called _kvmhost_ and
other hostgroups for guests.

Custom settings can be provided for each host or group of hosts in the
inventory.

To create a new group of guests to manage, create a new yml file under the
inventory directory. For example, if you wanted a set of guests for
OpenStack, you could create an openstack.yml file and populate it as required.

To manage specific hosts or groups, simply use Ansible's _--limit_ option to
specify the hosts or hostgroups (must also include _kvmhost_ group). This way
you can use the one inventory for lots of different guests and manage them
separately.

The KVM host is where the libvirt networks are created and therefore specified
as vars under that hostgroup.

Here is an example inventory file _simple.yml_ in YAML format. It is specifying
_kvmhost_ as localhost and creating three _simple_ CentOS guests, using the
role defaults.  Note there are two networks being created (_default_ and
_example_) and one which is removed (_other_).


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
simple:
  hosts:
    centos-simple-[0:2]:
      ansible_python_interpreter: /usr/bin/python
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
      - "example"
```

## Example Playbook

I've tried to keep the Ansible as a simple, logical set of steps and not get
too tricky. Having said that, the playbook is quite specific.

There are some tasks which can only be run on the KVM host and others in a
specific order. Some of the tasks need to be looped through with play_hosts but
only run once and delegated to the KVM host, like updating _/etc/hosts_ and
_~/.ssh/config_ on KVM host. Doing it this way avoids running in serial, which
is messy.

This code will also help by running a bunch of validation checks on the KVM
host and for your guest configs to try to catch anything that's not right.

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
curl -O https://cloud.centos.org/centos/7/images/CentOS-7-x86_64-GenericCloud.qcow2
sudo mv -iv CentOS-7-x86_64-GenericCloud.qcow2 /var/lib/libvirt/images/
```

### Run the playbook

Now run the Ansible playbook against the kvmhost and the guests in the _simple_
group (note that we limit to both the _kvmhost_ and the _simple_ hostgroups).

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
with --limit and pass in _virt_infra_state=undefined_ as a command line extra
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
