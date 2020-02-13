:tocdepth: 1

.. Please do not modify tocdepth; will be fixed when a new Sphinx theme is shipped.

.. note::

   **This technote is not yet published.**

   Bootstrapping the Deployment Platform

.. sectnum::

Introduction
============

.. TODO

This document was originally populated based on the Tucson core deployment, and
was fully expanded and verified in the Base DC deployment.

Requirements
============

Networking
^^^^^^^^^^

The following networks are required:

- A subnet where network services (IPA, DHCP, DNS) will be provisioned. This
  subnet should be assigned a range where reverse DNS records can be managed.

  .. note::
     At this time we're using Route53 to provide DNS for Cerro Pachon and
     La Serena. Because we have users connecting to the site via VPN and
     frequently use DNS servers from their home institutions, using publicly
     routable addresses and public DNS avoids a number of issues with resolution
     and reachability for those users.

The following networks are are preferred but not required:

- A subnet for in band host management interfaces. Reverse DNS records will
  also be created for these hosts.
- A subnet for out of band host management (e.g. IPMI, iDRAC, BMC, etc).

Hosts
=====

core1 hypervisor host
---------------------

.. Example given to indicate which version of CentOS we're deploying from.

.. code-block:: console
   $ curl -O http://mirror.netglobalis.net/centos/7.7.1908/isos/x86_64/CentOS-7-x86_64-Minimal-1908.iso
   $ sudo dd if=CentOS-7-x86_64-Minimal-1908.iso of=/dev/sdXXX status=progress


install OS
^^^^^^^^^^

Install centos from usb thumbdrive.

.. TODO develope kickstart file which can be used to consistently re-recreate
   the core 1 hypervisor.

Configure two physical interfaces per core hypervisor.  The first interface
(E.g. ``em1``) is the management/FQDN interface used to boot and access the
hypervisor itself.  The second interface (or bond) (E.g. ``em2``) is used to
pass vlans directly to VMs.  Unintentional filtering should be avoided on this
interface.

configure networking
^^^^^^^^^^^^^^^^^^^^

.. code-block:: yaml

  [jhoblitt@core1 network-scripts]$ ls -1 ifcfg-*
  ifcfg-br32
  ifcfg-br700
  ifcfg-br701
  ifcfg-br702
  ifcfg-br703
  ifcfg-br800
  ifcfg-br801
  ifcfg-em1
  ifcfg-em2
  ifcfg-em2.32
  ifcfg-em2.700
  ifcfg-em2.701
  ifcfg-em2.702
  ifcfg-em2.703
  ifcfg-em2.800
  ifcfg-em2.801
  ifcfg-lo
  ifcfg-p2p1
  ifcfg-p2p2
  [jhoblitt@core1 network-scripts]$ cat ifcfg-em1
  TYPE=Ethernet
  PROXY_METHOD=none
  BROWSER_ONLY=no
  BOOTPROTO=none
  IPV6INIT=no
  IPV6_AUTOCONF=no
  NAME=em1
  DEVICE=em1
  ONBOOT=yes
  IPADDR=140.252.35.7
  NETMASK=255.255.255.128
  GATEWAY=140.252.35.1
  [jhoblitt@core1 network-scripts]$ cat ifcfg-em2.32
  # File Managed by Puppet
  DEVICE="em2.32"
  BOOTPROTO="none"
  ONBOOT="yes"
  TYPE="none"
  USERCTL="no"
  PEERDNS="no"
  PEERNTP="no"
  VLAN="yes"
  BRIDGE="br32"
  [jhoblitt@core1 network-scripts]$ cat ifcfg-br32
  # File Managed by Puppet
  DEVICE="br32"
  BOOTPROTO="none"
  ONBOOT="yes"
  TYPE="bridge"
  USERCTL="no"
  PEERDNS="no"
  PEERNTP="no"

disable selinux
^^^^^^^^^^^^^^^

.. code-block:: console
   # sed -ie '/SELINUX=/s/=.*/=disabled/'
   # reboot

disable iptables
^^^^^^^^^^^^^^^^

.. code-block:: yaml

  yum install -y iptables-services
  systemctl stop iptables
  systemctl disable iptables
  iptables -F

create a dedicated volume for VM images
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

.. code-block:: yaml

  DEV=nvme0n1
  VOL=${DEV}p1
  parted -s /dev/${DEV} mklabel gpt
  parted -s /dev/${DEV} unit mib mkpart primary 1 100%
  parted -s /dev/${DEV} set 1 lvm on

  pvcreate /dev/${VOL}
  pvs
  vgcreate data /dev/${VOL}
  vgs
  lvcreate --size 500G --name vms data
  lvs

  mkfs.xfs /dev/data/vms

  echo "/dev/mapper/data-vms  /vm                     xfs     defaults        0 0
  " >> /etc/fstab
  mkdir /vm
  mount /vm

  # XXX figure out the correct ownership/permissions
  # vm images are owned qemu:qemu
  chmod 1777 /vm

install libvirt + extra tools
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

.. TODO figure out how to install with VNC instead of SPICE console to play
   nice[r] with foreman console redirection

.. code-block:: yaml

  yum install -y libvirt qemu-kvm
  yum install -y qemu-guest-agent qemu-kvm-tools virt-top virt-viewer libguestfs virt-who virt-what virt-install virt-manager

  systemctl enable libvirtd
  systemctl start libvirtd

  ### remove old default pool

  # enter virsh shell
  virsh

  pool-destroy default
  #pool-delete default
  pool-undefine default

  ### add new default pool at controlled path

  pool-define-as default dir - - - - "/vm"
  pool-start default
  pool-autostart default
  # sanity check
  pool-info default

  # exit virsh

  ### libvirt group

  sudo usermod --append --groups libvirt jhoblit

create foreman/puppet VM
^^^^^^^^^^^^^^^^^^^^^^^^

.. code-block:: yaml

  curl -O http://centos-distro.1gservers.com/7.7.1908/isos/x86_64/CentOS-7-x86_64-Minimal-1908.iso

  virt-install \
    --name=foreman \
    --vcpus=8 \
    --ram=16384 \
    --file-size=50 \
    --os-type=linux \
    --os-variant=rhel7 \
    --network bridge=br1621 \
    --location=/home/jhoblitt/CentOS-7-x86_64-Minimal-1908.iso

foreman/puppet VM
-----------------

disable selinux
^^^^^^^^^^^^^^^

disable iptables
^^^^^^^^^^^^^^^^

install foreman
^^^^^^^^^^^^^^^

.. code-block:: yaml

  sudo yum -y install https://yum.puppet.com/puppet6-release-el-7.noarch.rpm
  sudo yum -y install http://dl.fedoraproject.org/pub/epel/epel-release-latest-7.noarch.rpm
  sudo yum -y install https://yum.theforeman.org/releases/1.23/el7/x86_64/foreman-release.rpm
  sudo yum -y install foreman-installer

  foreman-installer \
    --enable-foreman-cli  \
    --enable-foreman-proxy \
    --foreman-proxy-tftp=true \
    --foreman-proxy-tftp-servername=140.252.32.218 \
    --foreman-proxy-dhcp=true \
    --foreman-proxy-dhcp-interface=eth1 \
    --foreman-proxy-dhcp-gateway=10.0.100.1 \
    --foreman-proxy-dhcp-nameservers="140.252.32.218" \
    --foreman-proxy-dhcp-range="10.0.100.50 10.0.100.60" \
    --foreman-proxy-dns=true \
    --foreman-proxy-dns-interface=eth0 \
    --foreman-proxy-dns-zone=tuc.lsst.cloud \
    --foreman-proxy-dns-reverse=100.0.10.in-addr.arpa \
    --foreman-proxy-dns-forwarders=140.252.32.21 \
    --foreman-proxy-foreman-base-url=https://foreman.tuc.lsst.cloud \
    --enable-foreman-plugin-remote-execution \
    --enable-foreman-plugin-dhcp-browser \
    --enable-foreman-proxy-plugin-remote-execution-ssh

  foreman-installer \
    --enable-foreman-cli \
    --enable-foreman-proxy \
    --foreman-proxy-tftp=true \
    --foreman-proxy-tftp-servername=139.229.162.45 \
    --foreman-proxy-dhcp=false \
    --foreman-proxy-dns=false \
    --foreman-proxy-foreman-base-url=https://foreman.cp.lsst.org \
    --enable-foreman-plugin-remote-execution \
    --enable-foreman-plugin-dhcp-browser \
    --enable-foreman-proxy-plugin-remote-execution-ssh

multi-homed network setup
^^^^^^^^^^^^^^^^^^^^^^^^^

Only applies to VMs with multiple interfaces.

.. code-block:: yaml

  [root@foreman settings.d]# sysctl -w net.ipv4.conf.all.arp_filter=1
  net.ipv4.conf.all.arp_filter = 1
  [root@foreman settings.d]# sysctl -w net.ipv4.conf.default.arp_filter=1
  net.ipv4.conf.default.arp_filter = 1

  cat > /etc/sysctl.d/91-rp_filter.conf <<END
  # allow response from interface, even if another interface is l2 reachable
  net.ipv4.conf.default.rp_filter = 0
  net.ipv4.conf.all.rp_filter = 0
  END

  cat > /etc/sysctl.d/92-arp_filter.conf <<END
  # allow multiple interfaces in same subnet
  net.ipv4.conf.default.arp_filter = 1
  net.ipv4.conf.all.arp_filter = 1
  END

  ### respond to foreman.tuc.lsst.cloud interface only via eth5

  [root@foreman ~]# cat /etc/sysconfig/network-scripts/ifcfg-eth5
  TYPE=Ethernet
  PROXY_METHOD=none
  BROWSER_ONLY=no
  BOOTPROTO=none
  DEFROUTE=yes
  IPV4_FAILURE_FATAL=no
  IPV6INIT=no
  IPV6_AUTOCONF=no
  NAME=eth5
  DEVICE=eth5
  ONBOOT=yes
  IPADDR=140.252.34.132
  NETMASK=255.255.255.192
  GATEWAY=140.252.34.129
  [root@foreman ~]# cat /etc/sysconfig/network-scripts/rule-eth5
  default via 140.252.34.129 table foreman
  140.252.34.128/26 dev eth5 table foreman

  [root@foreman ~]# cat /etc/iproute2/rt_tables
  #
  # reserved values
  #
  255	local
  254	main
  253	default
  0	unspec
  #
  # local
  #
  #1	inr.ruhep
  200	foreman

configure smart-proxy route53 plugin
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Install route53 plugin

.. code-block:: yaml

  yum install rubygem-smart_proxy_dns_route53

  [root@foreman ~]# cat /etc/foreman-proxy/settings.d/dns.yml
  :enabled: https
  :dns_ttl: 60

Configure AWS IAM policy (generally may be reused between all LSST foreman instances)

https://gist.github.com/jhoblitt/308d4069607d3237a4da4000c17eb5e3

Configure plugin

.. code-block:: yaml

  cat /etc/foreman-proxy/settings.d/dns_route53.yml
  #
  # Configuration file for 'dns_route53' DNS provider
  #
  
  # Set the following keys for the AWS credentials in use:
  :aws_access_key: ""
  :aws_secret_key: ""


if DNS resolution is blocked by firewall, change this foreman setting (via
foreman UI) to yes: ``Query local nameservers``

configure smart-proxy isc bind plugin (if not configured by foreman-installer)
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

.. code-block:: yaml

  yum install -y rubygem-smart_proxy_dhcp_remote_isc.noarch
  
  [root@foreman settings.d]# cat dhcp.yml 
  ---
  :enabled: https
  :use_provider: dhcp_isc
  :server: 127.0.0.1
  [root@foreman settings.d]# cat dhcp_isc.yml 
  ---
  #
  # Configuration file for ISC dhcp provider
  #
  
  :config: /etc/dhcp/dhcpd.conf
  :leases: /var/lib/dhcpd/dhcpd.leases
  
  # Redhat 5
  #
  #:config: /etc/dhcpd.conf
  #
  # Settings for Ubuntu
  #
  #:config: /etc/dhcp3/dhcpd.conf
  #:leases: /var/lib/dhcp3/dhcpd.leases
  
  # Specifies TSIG key name and secret
  
  #:key_name: secret_key_name
  #:key_secret: secret_key
  
  
  :omapi_port: 7911
  
  # use :server setting in dhcp.yml if you are managing a dhcp server which is not localhost
  

setup foreman libvirt integration with core1
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

See https://theforeman.org/manuals/1.23/index.html#5.2.5LibvirtNotes

.. code-block:: yaml

  yum install -y yum-utils augeas
  yum install -y foreman-libvirt
  
  su foreman -s /bin/bash
  ssh-keygen ....
  
  # on target libvirt host
  
  [root@core1 ~]# useradd -r -m foreman
  [root@core1 ~]# su - foreman
  [foreman@core1 ~]$ mkdir .ssh
  [foreman@core1 ~]$ chmod 700 .ssh
  [foreman@core1 .ssh]$ vi authorized_keys
  [foreman@core1 .ssh]$ chmod 600 authorized_keys
  
  # ensure polkit is being used for auth
  augtool -s set '/files/etc/libvirt/libvirtd.conf/access_drivers[1]' polkit
  
  # copied from fedora 30
  # /usr/share/polkit-1/rules.d/50-libvirt.rules
  
  cat << END > /etc/polkit-1/rules.d/80-libvirt.rules
  // Allow any user in the 'libvirt' group to connect to system libvirtd
  // without entering a password.
  
  polkit.addRule(function(action, subject) {
      if (action.id == "org.libvirt.unix.manage" &&
          subject.isInGroup("libvirt")) {
          return polkit.Result.YES;
      }
  });
  END
  
  systemctl restart libvirtd
  
  # sanity check
  su - foreman
  virsh --connect qemu:///system list --all
  
  # sanity check from foreman host
  sudo yum install -y libvirt-client
  su foreman -s /bin/bash
  virsh --connect qemu+ssh://foreman@core1.tuc.lsst.cloud/system list --all

boot strap puppet agent on core1
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

.. code-block:: yaml

  sudo yum -y install https://yum.puppet.com/puppet6-release-el-7.noarch.rpm
  sudo yum -y install http://dl.fedoraproject.org/pub/epel/epel-release-latest-7.noarch.rpm
  sudo yum -y install puppet-agent
  
  cat > /etc/puppetlabs/puppet/puppet.conf <<END
  
  
  [main]
  vardir = /opt/puppetlabs/puppet/cache
  logdir = /var/log/puppetlabs/puppet
  rundir = /var/run/puppetlabs
  ssldir = /etc/puppetlabs/puppet/ssl
  
  [agent]
  report          = true
  ignoreschedules = true
  ca_server       = foreman.tuc.lsst.cloud
  certname        = $(hostname -f)
  environment     = production
  server          = foreman.tuc.lsst.cloud
  END

foreman config for core1
^^^^^^^^^^^^^^^^^^^^^^^^

add host parameter:

role string hypervisor

Enable foreman-proxy bmc support
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

.. code-block:: yaml

  [root@foreman settings.d]# cat /etc/foreman-proxy/settings.d/bmc.yml 
  ---
  # BMC management (Bare metal power and bios controls)
  :enabled: true
  
  # Available providers:
  # - freeipmi / ipmitool - requires the appropriate package installed, and the rubyipmi gem
  # - shell - for local reboot control (requires sudo access to /sbin/shutdown for the proxy user)
  # - ssh - limited remote control (status, reboot, turn off)
  :bmc_default_provider: ipmitool
  
  systemctl restart foreman-proxy

foreman config
^^^^^^^^^^^^^^

global parameters

.. code-block:: yaml

  enable-puppetlabs-pc1-repo boolean true

hostgroup

coreXX

### parameters

cluster string core
site    string po

install r10k
^^^^^^^^^^^^

git is a r10k dep -- make sure it is installed.

.. code-block:: yaml

  sudo yum install -y git
  scl enable rh-ruby25 bash
  gem install r10k
  ln -s /opt/rh/rh-ruby25/root/usr/local/bin/r10k /usr/bin/r10k
  /opt/puppetlabs/puppet/bin/gem install r10k
  ln -sf /opt/puppetlabs/puppet/bin/r10k /usr/bin/r10k

r10k config
^^^^^^^^^^^

XXX put `r10k.yaml` some place it can be `curl`'d.

.. code-block:: yaml

  mkdir -p /etc/puppetlabs/r10k
  chown root:root /etc/puppetlabs/r10k
  chmod 0755 /etc/puppetlabs/r10k
  cat > /etc/puppetlabs/r10k/r10k.yaml <<END
  cachedir: "/var/cache/r10k"
  sources:
    control:
      remote: "https://github.com/lsst-it/lsst-itconf"
      basedir: "/etc/puppetlabs/code/environments"
    lsst_hiera_private:
      remote: "git@github.com:lsst-it/lsst-puppet-hiera-private.git"
      basedir: "/etc/puppetlabs/code/hieradata/private"
    lsst_hiera_public:
      remote: "https://github.com/lsst-it/lsst-puppet-hiera.git"
      basedir: "/etc/puppetlabs/code/hieradata/public"
  END
  chown root:root /etc/puppetlabs/r10k/r10k.yaml
  chmod 0644 /etc/puppetlabs/r10k/r10k.yaml

setup github deploy keys

.. code-block:: yaml

  cd /root/.ssh
  ssh-keygen -t rsa -b 2048 -C "foreman.cp.lsst.org" -f id_rsa -N ""

install public key on `lsst-it/lsst-puppt-hiera-private` repo:

https://github.com/lsst-it/lsst-puppet-hiera-private/settings/keys

use `hostname -f` as the title of the deploy key.

__Do not allow write access.__

pre-accept the github.com git hostkey

.. code-block:: yaml

  ssh-keyscan github.com >> ~/.ssh/known_hosts

run r10k

.. code-block:: yaml

  r10k deploy environment -ptv
