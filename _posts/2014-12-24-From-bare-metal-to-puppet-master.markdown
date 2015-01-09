---
layout: post
title:  "The beginning of the new infrastructure"
date:   2014-12-24 14:45:00
categories: puppet
tags:
  - CentOS7
  - Virsh
  - Libvirt
  - KVM
  - Virtualization
  - Puppet-Enterprise
---

Introduction
============
This is the story of my home computer infrastructure, and how I try to manage everything using the latest puppet enterprise, PE3.7.1.  Of course, I could have chosen the Open Source Version, but except for the puppet master/agent installation and configuration, how I will try to arrange my environments, modules, manifests, classes rspec files and everything else to get my home infrastructure properly managed by puppet, should also be usable with the Open Source Puppet Software.  At least, Now I don't have to document in too many pages how to install and configure my puppet master, since everything is handled bu the Puppet Enterprise Installation script.

I revived an old, out of support and unused DL380, which will be uses as my private hypervisor, where I will be hosting my KVM based virtual machines, and still have some resources left to get my hand on the revived container hype with the name 'Docker'.

But first things first, and we will have to solve the chicken/egg problem before we can start configuring everything with Puppet: Get a puppet master up and running.

The Hypervisor (dl380)
----------------------

For the initial installation, we will only configure one network interface, and we will do the  installation and configuration by hand. While we do the migration of my 'old' environment to this newborn one, we will need to do many adjustments, but then we will already have a working puppet infrastructure.

After putting in the CentOS7 netinstall cdrom into the drive I installed :

* Install Centos7 on my DL380, with 'Virtualization Host'
* Configured one interface manually with a static ip address
* Adjusted my dns server
* Post installation to obtain a working hypervisor.

  {% highlight shell-session %}
    [root@dl380 ~]# yum install qemu-kvm qemu-img
    [root@dl380 ~]# yum install virt-manager libvirt libvirt-python virt-install libvirt-client
  {% endhighlight %}

Preparing the second disk for the VM images
-------------------------------------------

I choose to use the LVM based approach, and wanted to refresh my  libvirt storage-pool commands.
Not all of my second disk is allocated to the Volume Group that will contain the guest images, so I
needed two partitions (the partition type code for LVM is 8e)

  {% highlight shell-session %}
    [root@dl380 ~]# fdisk -l /dev/sdb

    Disk /dev/sdb: 587.1 GB, 587128266752 bytes, 1146734896 sectors
    Units = sectors of 1 * 512 = 512 bytes
    Sector size (logical/physical): 512 bytes / 512 bytes
    I/O size (minimum/optimal): 512 bytes / 512 bytes
    Disk label type: dos
    Disk identifier: 0xa51c824c

    Device Boot      Start         End      Blocks   Id  System
    /dev/sdb1            2048   629147647   314572800   8e  Linux LVM
    /dev/sdb2       629147648  1146734895   258793624   83  Linux
  {% endhighlight %}

And create the initial volume group for the VM guest images

  {% highlight shell-session %}
    [root@dl380 ~]# vgcreate guest_images_lvm /dev/sdb1
      No device found for PV 6hvxiF-55oK-DTM2-Ub4F-XXiB-IXxD-UnQIMI.
      Physical volume "/dev/sdb1" successfully created
      Volume group "guest_images_lvm" successfully created
    [root@dl380 ~]#
    [root@dl380 ~]# vgs
      VG               #PV #LV #SN Attr   VSize   VFree
      centos_dl380       1   6   0 wz--n-  41.02g      0
      guest_images_lvm   1   0   0 wz--n- 300.00g 300.00g
    [root@dl380 ~]#
  {% endhighlight %}

Setting op a virsh storage pool
-------------------------------

Since we have already defined the VG, we can use following command :

  {% highlight shell-session %}
    [root@dl380 ~]# virsh pool-create-as --name guest_images_pool \
                                         --type logical \
                                         --source-dev=/dev/sdb1 \
                                         --source-name guest_images_lvm \
                                         --target /dev/guest_images_lvm
    Pool guest_images_pool created

    [root@dl380 ~]# virsh pool-list --all
    Name                 State      Autostart
    -----------------------------------------
    guest_images_pool    active     no
  {% endhighlight %}


We cannot turn the autostart on, because this pool is not persistent, because we did mot add the XML definition of it,
so we had to execute the next commands to solve the problem and activate the storage pool at boot-time :

  {% highlight shell-session %}
    [root@dl380 ~]# virsh pool-dumpxml guest_images_pool > /root/guest_images_pool.xml
    [root@dl380 ~]# virsh pool-define /root/guest_images_pool.xml
    Pool guest_images_pool defined from /root/guest_images_pool.xml

    [root@dl380 ~]# virsh  pool-info guest_images_pool
    Name:           guest_images_pool
    UUID:           c2e1baf1-c147-4875-8fa7-efe2c29629f3
    State:          running
    Persistent:     yes
    Autostart:      no
    Capacity:       300.00 GiB
    Allocation:     0.00 B
    Available:      300.00 GiB

    [root@dl380 ~]# virsh  pool-autostart guest_images_pool
    Pool guest_images_pool marked as autostarted

    [root@dl380 ~]# virsh  pool-info guest_images_pool
    Name:           guest_images_pool
    UUID:           c2e1baf1-c147-4875-8fa7-efe2c29629f3
    State:          running
    Persistent:     yes
    Autostart:      yes
    Capacity:       300.00 GiB
    Allocation:     0.00 B
    Available:      300.00 GiB
  {% endhighlight %}

And now we simply creates logical volumes using  libvirt. Again, this time we need to create at least one volume for
our first virtual machine, our initial puppet master. But we will look to manage this task (and all other needed steps
to create new virtual machines) by puppet.

  {% highlight shell-session %}
    [root@dl380 ~]# virsh vol-create-as guest_images_pool  pepuppet_guest_volume 30G
    Vol pepuppet_guest_volume created
    [root@dl380 ~]# virsh vol-list guest_images_pool
    Name                 Path
    -----------------------------------------
    pepuppet_guest_volume /dev/guest_images_lvm/pepuppet_guest_volume

    [root@dl380 ~]# lvs
      LV                    VG               Attr       LSize  Pool Origin Data%  Move Log Cpy%Sync Convert
      home                  centos_dl380     -wi-ao----  4.88g
      root                  centos_dl380     -wi-ao----  4.88g
      swap                  centos_dl380     -wi-ao----  6.84g
      tmp                   centos_dl380     -wi-ao----  4.88g
      usr                   centos_dl380     -wi-ao----  9.77g
      var                   centos_dl380     -wi-ao----  9.77g
      pepuppet_guest_volume guest_images_lvm -wi-a----- 30.00g
  {% endhighlight %}

And no, we are not ready to create our puppetmaster, since we need one way or another way to provide networking.
Since we will not only manage our virtual machines, but also physical machines, like some laptops and many others.
So we need to setup a bridged network.  In the first place we will create this bridge on the management network, currently
connected to my internal LAN. (later on we will adjust a lot of network stuff when my existing machines are migrated to my new setup)

The steps to create a bridge on a CentOS7 are :

* First check if the bridge module is loaded :

  {% highlight shell-session %}
    [root@dl380 ~]# modprobe --first-time bridge
    modprobe: ERROR: could not insert 'bridge': Module already in kernel
    [root@dl380 ~]#
  {% endhighlight %}

* Edit some files

  {% highlight shell-session %}
    [root@dl380 ~]# cd /etc/sysconfig/network-scripts/

    vi ifcfg-br0

      DEVICE=br0
      TYPE=Bridge
      IPADDR=192.168.10.70
      GATEWAY=192.168.10.10
      DNS1=192.168.20.1
      DOMAIN=koewacht.net
      PREFIX=24
      BOOTPROTO=none
      ONBOOT=yes
      DELAY=0

    vi ifcfg-enp2s0f0

      BOOTPROTO="none"
      DEFROUTE="yes"
      IPV4_FAILURE_FATAL="no"
      IPV6INIT="no"
      NAME="enp2s0f0"
      UUID="e83c276b-cfe0-4273-a3ef-08158c455609"
      ONBOOT="yes"
      HWADDR="00:26:55:4C:28:32"
      BRIDGE=br0
  {% endhighlight %}

* And a reboot to test the config properly. Yes we still can log in, so we do not need to attach a keyboard/monitor to the server for fixing. :

  {% highlight shell-session %}
    [root@dl380 ~]# brctl show
    br0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
            inet 192.168.10.70  netmask 255.255.255.0  broadcast 192.168.10.255
            inet6 fe80::226:55ff:fe4c:2832  prefixlen 64  scopeid 0x20<link>
            ether 00:26:55:4c:28:32  txqueuelen 0  (Ethernet)
            RX packets 51  bytes 7781 (7.5 KiB)
            RX errors 0  dropped 0  overruns 0  frame 0
            TX packets 52  bytes 7619 (7.4 KiB)
            TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

    enp2s0f0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
            ether 00:26:55:4c:28:32  txqueuelen 1000  (Ethernet)
            RX packets 41  bytes 7823 (7.6 KiB)
            RX errors 0  dropped 0  overruns 0  frame 0
            TX packets 43  bytes 6816 (6.6 KiB)
            TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

    [root@dl380 ~]# brctl show
    bridge name     bridge id             STP enabled   interfaces
    br0             8000.0026554c2832     no            enp2s0f0
    virbr0          8000.525400086dc4     yes           virbr0-nic
    [root@dl380 ~]#
  {% endhighlight %}

The preparation is almost done, but since we don't like to click through the installation screens,
we just going to create a basic 'Kickstart' files to automate this boring process :

  {% highlight shell-session %}
    #version=RHEL7
    # System authorization information
    auth --enableshadow --passalgo=sha512

    # Use network installation
    url --mirrorlist="http://mirrorlist.centos.org/?release=7&arch=x86_64&repo=os"
    # Run the Setup Agent on first boot
    firstboot --enable
    #ignoredisk --only-use=sda
    # Keyboard layouts
    keyboard --vckeymap=us --xlayouts='us'
    # System language
    lang en_US.UTF-8

    # Network information
    network  --device=eth0 --bootproto=static --gateway=192.168.10.10 --ip=192.168.10.130 --nameserver=192.168.20.1 --netmask=255.255.255.0 --noipv6 --activate --hostname=pepuppet.koewacht.net
    # Root password
    rootpw --iscrypted <some big hash thing>
    # System timezone
    timezone Europe/Brussels --isUtc
    user --groups=wheel --name=johan --password=<some big hash thing> --iscrypted
    # System bootloader configuration
    bootloader --location=mbr
    # Partition clearing information
    clearpart --all --initlabel
    autopart --type=lvm

    %packages
    @base
    @core

    %end
  {% endhighlight %}

* And now we are ready to install our first VM on our old brand new hypervisor, and  watch some magic how a login prompt is born.


  {% highlight shell-session %}
    [root@dl380 ~]# virt-install \
      --name=pepuppet \
      --controller type=scsi,model=virtio-scsi \
      --disk vol=guest_images_pool/pepuppet_guest_volume,device=disk,bus=scsi \
      --graphics=none \
      --vcpus=2 \
      --ram=6144 \
      --network bridge=br0 \
      --os-type=linux \
      --os-variant=rhel7 \
      --cpu=host \
      --description="Puppet Enterprise 3.7.x VM" \
      --location="http://mirror.centos.org/centos-7/7/os/x86_64/" \
      --initrd-inject=/root/pepuppet.ks --extra-args "ks=file:/pepuppet.ks console=ttyS0,115200n8 serial"
    [root@dl380 ~]#
  {% endhighlight %}

At this stage, We have a hypervisor, and out first guest running, which will become our puppetmaster.
Lets get PE371 installed :

We downloaded the software from puppetlabs, and transferred it to the root directory of the newly created VM.
Here is the summary of the installation procedure, again I prefer the CLI based one, using the answer file method.

  {% highlight shell-session %}
    [root@pepuppet ~]# tar -xzf puppet-enterprise-3.7.1-el-7-x86_64.tar.gz
    [root@pepuppet ~]# ln -sf puppet-enterprise-3.7.1-el-7-x86_64 puppet-enterprise
    [root@pepuppet puppet-enterprise]# touch dummy.txt
    [root@pepuppet puppet-enterprise]# ./puppet-enterprise-installer -A dummy.txt -s pepuppet_answer.txt
    =================================================================================================================

    Puppet Enterprise v3.7.1 installer

    Puppet Enterprise documentation can be found at http://docs.puppetlabs.com/pe/3.7/

    -----------------------------------------------------------------------------------------------------------------

    STEP 1: READ ANSWERS FROM FILE

    ## Reading answers from file: ./dummy.txt

    -----------------------------------------------------------------------------------------------------------------

    STEP 2: SELECT AND CONFIGURE ROLES

       This installer lets you select and install the various roles required in a Puppet Enterprise deployment:
       puppet master, console, database, cloud provisioner, and puppet agent.

    NOTE: when specifying hostnames during installation, use the fully-qualified domain name (foo.example.com) rather than a shortened name (foo).


    -> puppet master

       The puppet master serves configurations to a group of puppet agent nodes. This role also provides
       MCollective's message queue and client interface. It should be installed on a robust, dedicated server.

    ?? Install puppet master? [y/N] y

    -> standalone install

       You may choose to either install PuppetDB and the console on this node, or to install each service on its
       own node. If you choose not to install PuppetDB and the console on this node, you will be asked where to
       find them.

    ?? Install PuppetDB and console on this node? [Y/n]

    -> cloud provisioner

       The cloud provisioner can create and bootstrap new machine instances and add them to your Puppet
       infrastructure. It should be installed on a trusted node where site administrators have shell access.

    ?? Install the cloud provisioner? [y/N]

    -> puppet agent

       The puppet agent role is automatically installed with the console, puppet master, puppetdb, and cloud
       provisioner roles.

    ?? The puppet master's certificate will contain a unique name ("certname"); this should be the main DNS name
       at which it can be reliably reached. Puppet master's certname? [Default: pepuppet.koewacht.net]
    ?? The puppet master's certificate can contain DNS aliases; agent nodes will only trust the master if they
       reach it at its certname or one of these official aliases. Puppet master's DNS aliases (comma-separated
       list)? [Default: pepuppet,pepuppet.koewacht.net,puppet,puppet.koewacht.net]

       The Puppet Enterprise console and PuppetDB require a PostgreSQL database and a user account able to edit
       it. Puppet Enterprise includes a Postgresql server which you can install locally, or you can specify an
       existing remote database (which must be configured and available prior to installing the console or
       PuppetDB).

    ?? Install the included Puppet Enterprise PostgreSQL server locally? [Y/n]
    ?? What is a port for use by the PE console? [Default: 443]
    ?? Password for Puppet Enterprise Console superuser 'admin' (minimum 8 characters)?
    Confirm Password:

    -> Vendor Packages

       Puppet Enterprise may require additional packages from your operating system vendor. You will need to
       either install these yourself, or allow them to be automatically installed from your operating system's
       package repositories.

       Additional vendor packages required for installation:
       * libxslt
       * mailcap


    ?? Allow automatic installation of these packages? [Y/n]

    -----------------------------------------------------------------------------------------------------------------

    STEP 3: CONFIRM PLAN

    You have selected to install the following components (and their dependencies)
    * Puppet Master
    * PuppetDB
    * Console
    * Puppet Agent

    ?? Perform installation? [Y/n]

    -----------------------------------------------------------------------------------------------------------------

    STEP 4: SAVE ANSWERS

    ## Answers saved in the following file: pepuppet_answer.txt

    =================================================================================================================
    [root@pepuppet puppet-enterprise]# ./puppet-enterprise-installer -a pepuppet_answer.txt
  {% endhighlight %}

And that's all there is to get a full blown puppet master, console, puppetdb and a working mcollective
(and having some patience for the installation to finish).

I did have some issues and had to do the installation over :

* the hostname must be resolvable.  I still had an old entry in my dns, pointing to a different ip address,
which made the installation failed, during pe-console startup.

Since running PE, we need to remember everything is slightly different organised :

* installed to "/opt/puppet"
* configuration in "/etc/puppetlabs"
* auto-generated database users and passwords have been saved to '/etc/puppetlabs/installer/database_info.install'

Since we have the firewall on every server running,  and SElinux is of course also active, we need to check the proper working of everything.

Normally we manage those things with puppet, but I'm not yet sure if we are already in the chicken, or still in  the egg stage.
So we still manage those rules manually. And it is in the meantime a good introduction to the new firewall configuration utilities in CentOS7.
In this initial setup, the eth0 interface is assigned to the public zone.
One can look up the zone of an interface with :

  {% highlight shell-session %}
    [root@pepuppet puppet-enterprise]# firewall-cmd --get-zone-of-interface=eth0
    public
  {% endhighlight %}

And the current setting of this zone can be viewed using :

  {% highlight shell-session %}
    [root@pepuppet puppet-enterprise]# firewall-cmd --zone=public --list-all
    public (default, active)
      interfaces: eth0
      sources:
      services: dhcpv6-client ssh
      ports:
      masquerade: no
      forward-ports:
      icmp-blocks:
      rich rules:
  {% endhighlight %}


We have to add the puppet TCP ports to this zone, which are : 443, 4433, 4435, 8140, 61613
One way to do this is to define a new puppet firewall service :

we create a xml file with following content :/etc/firewalld/services/puppet-enterprise.xml
  {% highlight xml %}
    <?xml version="1.0" encoding="utf-8"?>
      <service>
        <short>Puppet monolitic</short>
        <description>Firewall configuration for a Puppet Enterprise monolithic installation.</description>
        <port port="443"   protocol="tcp"/>
        <port port="4433"  protocol="tcp"/>
        <port port="4435"  protocol="tcp"/>
        <port port="8140"  protocol="tcp"/>
        <port port="61613" protocol="tcp"/>
      </service>
  {% endhighlight %}

and adjust SElinux settings :

  {% highlight shell-session %}
    [root@pepuppet services]# ls -lZ puppet-enterprise.xml
    -rw-r--r--. root root unconfined_u:object_r:firewalld_etc_rw_t:s0 puppet-enterprise.xml
    [root@pepuppet services]# ls -lZ /usr/lib/firewalld/services/dns.xml
    -rw-r-----. root root system_u:object_r:lib_t:s0       /usr/lib/firewalld/services/dns.xml
    [root@pepuppet services]# chcon --reference /usr/lib/firewalld/services/dns.xml puppet-enterprise.xml
    [root@pepuppet services]# ls -lZ puppet-enterprise.xml
    -rw-r--r--. root root system_u:object_r:lib_t:s0       puppet-enterprise.xml
  {% endhighlight %}

Now we are ready to open the ports on the firewall :

  {% highlight shell-session %}
    [root@pepuppet services]# firewall-cmd --zone=public --add-service=puppet-enterprise --permanent
    success
    [root@pepuppet services]# firewall-cmd --reload
    success
    [root@pepuppet services]#
  {% endhighlight %}

Ans we verify this with the command :

  {% highlight shell-session %}
    [root@pepuppet]#iptables -L -n
    <....>
    Chain IN_public_allow (1 references)
    target     prot opt source               destination
    ACCEPT     tcp  --  0.0.0.0/0            0.0.0.0/0            tcp dpt:443 ctstate NEW
    ACCEPT     tcp  --  0.0.0.0/0            0.0.0.0/0            tcp dpt:4433 ctstate NEW
    ACCEPT     tcp  --  0.0.0.0/0            0.0.0.0/0            tcp dpt:4435 ctstate NEW
    ACCEPT     tcp  --  0.0.0.0/0            0.0.0.0/0            tcp dpt:8140 ctstate NEW
    ACCEPT     tcp  --  0.0.0.0/0            0.0.0.0/0            tcp dpt:61613 ctstate NEW
    <....>
  {% endhighlight %}

and try to connect to the puppet console  to verify the working, by surfing to the following URL :

    https://pepuppet.koewacht.net

The next step is to install and configure r10k and prepare everything to implement the profiles/role pattern.
But that's for another post.

