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

Prepare host hardware
^^^^^^^^^^^^^^^^^^^^^

Boot up all core nodes and perform the following configuration changes.

1. BIOS settings -> Security settings -> Power recovery -> ON.

   .. note::

      It is absolutely essential that core nodes boot on power restore, as
      these systems will provide the services needed to interact with OOB
      management on other hosts. We commonly apply this same configuration
      to all servers but this is a convention.

2. BIOS settings -> Miscellaneous settings -> F1/F2 on error -> disable.
3. Device settings -> PERC -> Create a mirrored RAID array. Prefer 1TB or smaller drives.

core1 hypervisor host
---------------------

.. Example given to indicate which version of CentOS we're deploying from.

.. code-block:: bash

   curl -O http://mirror.netglobalis.net/centos/7.7.1908/isos/x86_64/CentOS-7-x86_64-Minimal-1908.iso
   sudo dd if=CentOS-7-x86_64-Minimal-1908.iso of=/dev/sdXXX status=progress

Install OS on core1
^^^^^^^^^^^^^^^^^^^

Install Centos from USB thumbdrive.

1. Partitioning: use default partition table.
2. Network: set hostname to core1.<domain>
3. Timezone: Etc/UTC (Will be managed by Puppet when the core cluster is live.)
4. Root password: see the password vault for the root password. (Will be managed
   by Puppet when the core cluster is live.)

.. TODO develope kickstart file which can be used to consistently re-recreate
   the core 1 hypervisor.

Configure two physical interfaces per core hypervisor.  The first interface
(E.g. ``em1``) is the management/FQDN interface used to boot and access the
hypervisor itself.  The second interface (or bond) (E.g. ``em2``) is used to
pass vlans directly to VMs.  Unintentional filtering should be avoided on this
interface.

Configure networking
^^^^^^^^^^^^^^^^^^^^

Configure the host management interface.

.. code-block:: bash

   nmcli con edit em1
   set connection.autoconnect yes
   set ipv4.addresses 139.229.135.2/24
   set ipv4.dns 139.229.136.35
   set ipv4.gateway 139.229.135.254
   save
   activate
   exit

Configure the hypervisor interface as a trunk. VMs will be attached to subinterfaces.

.. code-block:: bash

   # If the interface is already defined
   nmcli con modify p2p1 connection.autoconnect yes
   # If the interface needs to be created
   nmcli con add save yes type ethernet ifname p2p1 con-name p2p1 \
      connection.autoconnect yes ipv4.method disabled ipv6.method ignore

Create a bridge for VMs on VLAN 1800.

.. code-block:: bash

   VLAN=1800
   nmcli conn add save yes type bridge ifname br${VLAN} con-name br${VLAN} \
      connection.autoconnect yes ipv4.method disabled ipv6.method ignore

Attach the VLAN ${VLAN} subinterfaces to the bridge.

.. code-block:: bash

   VLAN=1800
   nmcli con add save yes type vlan dev p2p1 id ${VLAN} con-name p2p1.${VLAN} \
      connection.slave-type bridge connection.master br${VLAN} connection.autoconnect yes \

The resulting ifcfg scripts should resemble the following:

.. code-block:: console

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

Disable SELinux
^^^^^^^^^^^^^^^

.. code-block:: bash

   sed -ie '/SELINUX=/s/=.*/=disabled/' /etc/selinux/config
   # Perform a fast reboot - don't reinitialize the hardware.
   systemctl kexec

Disable iptables
^^^^^^^^^^^^^^^^

.. code-block:: bash

   yum install -y iptables-services
   systemctl stop iptables
   systemctl disable iptables
   iptables -F

Create a dedicated volume for VM images
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

.. code-block:: bash

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

  echo "/dev/mapper/data-vms  /vm                     xfs     defaults        0 0" >> /etc/fstab
  mkdir /vm
  mount /vm

  # XXX figure out the correct ownership/permissions
  # vm images are owned qemu:qemu
  chmod 1777 /vm

Install libvirt + extra tools
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

.. TODO figure out how to install with VNC instead of SPICE console to play
   nice[r] with foreman console redirection

.. code-block:: bash

  yum install -y libvirt qemu-kvm
  yum install -y qemu-guest-agent qemu-kvm-tools virt-top \
                 virt-viewer libguestfs virt-who virt-what \
                 virt-install virt-manager

  systemctl enable libvirtd
  systemctl start libvirtd

  ### remove old default pool
  virsh pool-destroy default
  virsh pool-undefine default

  ### add new default pool at controlled path

  virsh pool-define-as --name default --type dir - - - - "/vm"
  virsh pool-start default
  virsh pool-autostart default
  # sanity check
  virsh pool-info default

  ### libvirt group

  sudo usermod --append --groups libvirt jhoblit

Create foreman/puppet VM
^^^^^^^^^^^^^^^^^^^^^^^^

.. code-block:: bash

   curl -O http://centos-distro.1gservers.com/7.7.1908/isos/x86_64/CentOS-7-x86_64-Minimal-1908.iso
   VLAN=1800
   virt-install \
     --name=foreman \
     --vcpus=8 \
     --ram=16384 \
     --file-size=50 \
     --os-type=linux \
     --os-variant=rhel7 \
     --network bridge=br${VLAN} \
     --location=/tmp/CentOS-7-x86_64-Minimal-1908.iso

Foreman/puppet VM
-----------------

Disable SELinux
^^^^^^^^^^^^^^^

.. code-block:: bash

   sed -ie '/SELINUX=/s/=.*/=disabled/' /etc/selinux/config
   # Perform a fast reboot - don't reinitialize the hardware.
   systemctl kexec

Disable iptables
^^^^^^^^^^^^^^^^

.. code-block:: bash

   yum install -y iptables-services
   systemctl stop iptables
   systemctl disable iptables
   iptables -F

install foreman
^^^^^^^^^^^^^^^

.. code-block:: bash
   FOREMAN_VERSION="1.24"
   sudo yum -y install https://yum.puppet.com/puppet6-release-el-7.noarch.rpm
   sudo yum -y install http://dl.fedoraproject.org/pub/epel/epel-release-latest-7.noarch.rpm
   sudo yum -y install https://yum.theforeman.org/releases/"${FOREMAN_VERSION}"/el7/x86_64/foreman-release.rpm
   sudo yum -y install foreman-installer

Tucson:

.. code-block:: bash

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

Cerro Pachon:

.. code-block:: bash

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

BDC:

.. code-block:: bash

   #CORE1
   systemctl disable --now firewalld
   FOREMAN_IP="139.229.135.5"
   DHCP_RANGE="139.229.135.192 139.229.135.253"
   DHCP_GATEWAY="139.229.135.254"
   DHCP_NAMESERVERS="139.229.136.35"
   DNS_ZONE="ls.lsst.org"
   DNS_REVERSE_ZONE="135.229.139.in-addr.arpa"
   DNS_FORWARDERS="139.229.136.35"
   FOREMAN_URL="https://foreman.ls.lsst.org"
   sudo foreman-installer \
     --enable-foreman-cli  \
     --enable-foreman-proxy \
     --foreman-proxy-tftp=true \
     --foreman-proxy-tftp-servername="${FOREMAN_IP}" \
     --foreman-proxy-dhcp=true \
     --foreman-proxy-dhcp-interface=eth0 \
     --foreman-proxy-dhcp-gateway="${DHCP_GATEWAY}" \
     --foreman-proxy-dhcp-nameservers="${DHCP_NAMESERVERS}" \
     --foreman-proxy-dhcp-range="${DHCP_RANGE}" \
     --foreman-proxy-dns=true \
     --foreman-proxy-dns-interface=eth0 \
     --foreman-proxy-dns-zone="${DNS_ZONE}" \
     --foreman-proxy-dns-reverse="${DNS_REVERSE_ZONE}" \
     --foreman-proxy-dns-forwarders="${DNS_FORWARDERS}" \
     --foreman-proxy-foreman-base-url="${FOREMAN_URL}" \
     --enable-foreman-plugin-remote-execution \
     --enable-foreman-plugin-dhcp-browser \
     --enable-foreman-proxy-plugin-remote-execution-ssh
   virsh autostart foreman



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
  :use_provider: dns_route53

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

  [root@foreman]# cat /etc/foreman-proxy/settings.d/dhcp.yml
  ---
  :enabled: https
  :use_provider: dhcp_isc
  :server: 127.0.0.1
  [root@foreman]# cat /etc/foreman-proxy/settings.d/dhcp_isc.yml
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
##Should be puppetize in the near future

.. code-block:: yaml

  ##On target libvirt host (core1)
  [root@core1 ~]# useradd -r -m foreman
  [root@core1 ~]# usermod -a -G libvirt foreman
  [root@core1 ~]# su - foreman
  [foreman@core1 ~]$ mkdir .ssh
  [foreman@core1 ~]$ chmod 700 .ssh

  ##On Foreman instance
  yum install -y yum-utils augeas foreman-libvirt libvirt-client
  su foreman -s /bin/bash
  ssh-keygen
  scp -l root /usr/share/foreman/.ssh/id_rsa.pub root@core1.ls.lsst.org:/home/foreman/.ssh/authorized_keys

  #Again on target libvirt host (core1)
  [foreman@core1 .ssh]$ chmod 600 authorized_keys

  # ensure polkit is being used for auth
  augtool -s set '/files/etc/libvirt/libvirtd.conf/access_drivers[1]' polkit

  # copied from fedora 30
  # /usr/share/polkit-1/rules.d/50-libvirt.rules

  ## Important! The commented "if" and the AdminRule must be solved
  cat << END > /etc/polkit-1/rules.d/80-libvirt.rules
  // Allow any user in the 'libvirt' group to connect to system libvirtd
  // without entering a password.

  polkit.addRule(function(action, subject) {
      //if (action.id == "org.libvirt.unix.manage" &&
      if (subject.isInGroup("libvirt")) {
          return polkit.Result.YES;
      }
  });

  polkit.addAdminRule(function(action, subject) {
      return ["unix-group:libvirt"];

  END

  systemctl restart libvirtd polkit

  # sanity check from core1
  su - foreman
  virsh --connect qemu:///system list --all

  # sanity check from foreman host
  sudo yum install -y libvirt-client
  su foreman -s /bin/bash
  virsh --connect qemu+ssh://foreman@core1.tuc.lsst.cloud/system list --all

boot strap puppet agent on core1
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

.. code-block:: yaml

  ##At core1
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
  ca_server       = foreman.ls.lsst.org
  certname        = $(hostname -f)
  environment     = production
  server          = foreman.ls.lsst.org

  END

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

Install and configure r10k
^^^^^^^^^^^^^^^^^^^^^^^^^^

.. code-block:: yaml

  # git is a r10k dep -- make sure it is installed.
  sudo yum install -y git
  scl enable rh-ruby25 bash
  gem install r10k
  ln -s /opt/rh/rh-ruby25/root/usr/local/bin/r10k /usr/bin/r10k
  /opt/puppetlabs/puppet/bin/gem install r10k
  ln -sf /opt/puppetlabs/puppet/bin/r10k /usr/bin/r10k

.. code-block:: yaml

  install -d -m 0755 -o root -g root /etc/puppetlabs/r10k
  install -m 0644 -o root -g root /dev/stdin /etc/puppetlabs/r10k/r10k.yaml <<END
  cachedir: "/var/cache/r10k"
  sources:
    control:
      remote: "https://github.com/lsst-it/lsst-itconf"
      basedir: "/etc/puppetlabs/code/environments"
    lsst_hiera_private:
      remote: "git@github.com:lsst-it/lsst-puppet-hiera-private.git"
      basedir: "/etc/puppetlabs/code/hieradata/private"
  END

Setup GitHub deploy keys and GitHub SSH known hosts:

.. code-block:: yaml

  install -d -m 0700 -o root -g root /root/.ssh
  cd /root/.ssh
  ssh-keygen -t rsa -b 2048 -C "$(hostname -f) - r10k github" -f id_rsa -N ""
  # pre-accept the github.com git hostkey
  ssh-keyscan github.com >> ~/.ssh/known_hosts

Install public key on `lsst-it/lsst-puppt-hiera-private` repo:

https://github.com/lsst-it/lsst-puppet-hiera-private/settings/keys


__Do not allow write access.__

Run r10k to populate the Puppet code, and then import all environments into Foreman.

.. code-block:: yaml

  r10k deploy environment -ptv
  hammer proxy import-classes --id 1

Foreman configuration
=====================

Global parameters
^^^^^^^^^^^^^^^^^

.. code-block:: bash

   # Configure the Foreman site and organization.
   hammer global-parameter set --name org --parameter-type string --value lsst
   hammer global-parameter set --name site --parameter-type string --value ls
   # Configure the Foreman site and organization.
   hammer global-parameter set --name enable-puppetlabs-puppet6-repo --parameter-type boolean --value true

Hostgroup dependencies
^^^^^^^^^^^^^^^^^^^^^^

Generate domains, subnets, and other resources that will be associated with
hosts and hostgroups.

.. code-block:: bash

   # Update the `ls.lsst.org` domain to use Foreman as the forward DNS proxy
   hammer domain update --name ls.lsst.org --dns foreman.ls.lsst.org

   # Define a subnet for core services, using the Foreman smart proxies.
   # Note that the `--dns` option sets the reverse DNS smart proxy.
   hammer subnet create --name IT-Services \
      --network-type 'IPv4' --boot-mode DHCP \
      --network 139.229.135.0 --mask 255.255.255.0 \
      --gateway 139.229.135.254 \
      --dns-primary 139.229.136.35 \
      --from 139.229.135.1 --to 139.229.135.32 --ipam DHCP \
      --domains ls.lsst.org \
      --tftp foreman.ls.lsst.org --dns foreman.ls.lsst.org --dhcp foreman.ls.lsst.org

   # Create a default partition table that sets up a simple partition table on the
   # first disk.
   hammer partition-table create --name "Kickstart sda only" \
      --description "Kickstart sda only" \
      --os-family "Redhat" --operatingsystems "CentOS 7.7.1908" \
      --file /dev/stdin <<-END
   <%#
   kind: ptable
   name: Kickstart default
   model: Ptable
   oses:
   - CentOS
   - Fedora
   - RedHat
   %>
   ignoredisk --only-use=sda
   zerombr
   clearpart --all --initlabel
   autopart <%= host_param('autopart_options') %>
   END

   # Associate the default kickstart partitioning table as well - we'll use that for libvirt VMs.
   hammer partition-table add-operatingsystem --name 'Kickstart default' --operatingsystem 'CentOS 7.7.1908'

   # Installation media and operating system versions need to be associated, and
   # we need a medium defined to create the `ls/corels` hostgroup. Create that
   # association here.
   hammer medium add-operatingsystem --name "CentOS mirror" --operatingsystem "CentOS 7.7.1908"

TODO: provisioning template/operating system associations

.. code-block:: bash

   # Scan for all templates associated with CentOS
   for i in {1..200}; do
     hammer template info --id $i \
       | ruby -e 'str = ARGF.read; puts str if str =~ /CentOS/'
   done

.. code-block:: bash

   hammer os add-config-template --config-template "Kickstart default" --title "CentOS 7.7.1908"
   hammer os add-config-template --config-template "Kickstart default iPXE" --title "CentOS 7.7.1908"
   hammer os add-config-template --config-template "Kickstart default PXEGrub2" --title "CentOS 7.7.1908"
   hammer os add-config-template --config-template "CloudInit default" --title "CentOS 7.7.1908"
   hammer os add-config-template --config-template "UserData open-vm-tools" --title "CentOS 7.7.1908"

   # TODO: replace hardcoded OS ID
   hammer os set-default-template --id 1 \
     --config-template-id "$(hammer template info --name 'Kickstart default' --fields id | awk '{ print $2 }')"
   hammer os set-default-template --id 1 \
     --config-template-id "$(hammer template info --name 'Kickstart default iPXE' --fields id | awk '{ print $2 }')"
   hammer os set-default-template --id 1 \
     --config-template-id "$(hammer template info --name 'Kickstart default PXEGrub2' --fields id | awk '{ print $2 }')"

Hostgroups
^^^^^^^^^^

Create hostgroups for the entire site (e.g. ``ls``) and the core group (e.g.
``ls/corels``). The group for the entire site is needed to set reasonable
provisioning defaults.

.. code-block:: bash

   hammer hostgroup create \
      --name ls \
      --description "All La Serena hosts" \
      --puppet-ca-proxy foreman.ls.lsst.org \
      --puppet-proxy foreman.ls.lsst.org
   HOSTGROUP="corels"
   hammer hostgroup create \
      --name "${HOSTGROUP}" \
      --description "Core services for La Serena" \
      --parent ls \
      --puppet-environment corels_production \
      --architecture x86_64 \
      --domain ls.lsst.org \
      --subnet IT-Services \
      --operatingsystem "CentOS 7.7.1908" \
      --medium "CentOS mirror" \
      --partition-table "Kickstart sda only" \
      --group-parameters-attributes '[{"name": "cluster", "value": '"${HOSTGROUP}"', "parameter_type": "string"}]'

   hammer hostgroup create --name vm --parent corels \
      --compute-profile 1-Small --partition-table 'Kickstart default' \
      --pxe-loader 'iPXE Embedded'

Host classification
^^^^^^^^^^^^^^^^^^^

Reclassify the foreman and core nodes.

.. code-block:: bash

   hammer host update --name foreman.ls.lsst.org --parameters role=foreman --hostgroup-title ls/corels
   hammer host update --name core1.ls.lsst.org --parameters role=hypervisor --hostgroup-title ls/corels

Adding hypervisors
^^^^^^^^^^^^^^^^^^

.. code-block:: bash

   SHORTNAME=core2
   BRIDGE=br1800
   hammer compute-resource create --name "${SHORTNAME}" \
     --display-type VNC --provider Libvirt \
     --url "qemu+ssh://foreman@$SHORTNAME.ls.lsst.org/system"

   hammer compute-profile values update \
     --compute-profile 1-Small --compute-resource "${SHORTNAME}" \
     --compute-attributes "cpus=2,memory=$((4 * 1024 * 1024 * 1024))" \
     --interface "compute_type=bridge,compute_bridge=${BRIDGE},compute_model=virtio" \
     --volume "pool_name=default,capacity=40G,allocation=0,format_type=raw"

   hammer compute-profile values update \
     --compute-profile 2-Medium --compute-resource "${SHORTNAME}" \
     --compute-attributes "cpus=4,memory=$((8 * 1024 * 1024 * 1024))" \
     --interface "compute_type=bridge,compute_bridge=${BRIDGE},compute_model=virtio" \
     --volume "pool_name=default,capacity=80G,allocation=0,format_type=raw"

   hammer compute-profile values update \
     --compute-profile 3-Large --compute-resource "${SHORTNAME}" \
     --compute-attributes "cpus=8,memory=$((16 * 1024 * 1024 * 1024))" \
     --interface "compute_type=bridge,compute_bridge=${BRIDGE},compute_model=virtio" \
     --volume "pool_name=default,capacity=160G,allocation=0,format_type=raw"
