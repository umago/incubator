VM (Ubuntu and Fedora)
==============

(There are detailed instructions available below, the overview and
configuration sections provide background information).

Overview:
* Setup SSH access to let the seed node turn on/off other libvirt VMs.
* Setup a VM that is your seed node
* Setup N VMs to pretend to be your cluster
* Go to town testing deployments on them.
* For troubleshooting see [troubleshooting.md](troubleshooting.md)
* For generic deployment information see [Deploying.md](Deploying.md)

Configuration
-------------

The seed instance expects to run with its eth0 connected to the outside world,
via whatever IP range you choose to setup. You can run NAT, or not, as you
choose. This is how we connect to it to run scripts etc - though you can
equally log in on its console if you like.

We use flat networking with all machines on one broadcast domain for dev-test.

The eth1 of your seed instance should be connected to your bare metal cloud
LAN. The seed VM uses the rfc5735 TEST-NET-1 range - 192.0.2.0/24 for
bringing up nodes, and does its own DHCP etc, so do not connect it to a network
shared with other DHCP servers or the like. The instructions in this document
create a bridge device ('br99') on your machine to emulate this with virtual
machine 'bare metal' nodes.


  NOTE: We recommend using an apt/HTTP proxy and setting the http_proxy
        environment variable accordingly in order to speed up the image build
        times.  See footnote [3] to set up Squid proxy.

  NOTE: The CPU architecture specified in several places must be consistent.
  	The examples here use 32-bit arch for the reduced memory footprint.  If
	you are running on real hardware, or want to test with 64-bit arch,
	replace i386 => amd64 and i686 => x86_64 in all the commands below. You
	will of course need amd64 capable hardware to do this.

Detailed instructions
---------------------

__(Note: all of the following commands should be run on your host machine, not inside the seed VM)__

1. Before you start, check to see that your machine supports hardware
   virtualization, otherwise performance of the test environment will be poor.
   We are currently bringing up an LXC based alternative testing story, which
   will mitigate this, thoug the deployed instances will still be full virtual
   machines and so performance will be significantly less there without
   hardware virtualisation.

1. Also check ssh server is running on the host machine and port 22 is open for
   connections from virbr0 -  VirtPowerManager will boot VMs by sshing into the
   host machine and issuing libvirt/virsh commands. The user these instructions
   use is your own, but you can also setup a dedicated user if you choose.

1. Choose a base location to put all of the source code.

        mkdir ~/tripleo
        export TRIPLEO_ROOT=~/tripleo
        cd $TRIPLEO_ROOT

1. git clone this repository to your local machine.

        git clone https://github.com/tripleo/incubator.git

1. git clone bm_poseur to your local machine.

        git clone https://github.com/tripleo/bm_poseur.git

1. git clone diskimage-builder and the tripleo elements likewise.

        git clone https://github.com/stackforge/diskimage-builder.git
        git clone https://github.com/stackforge/tripleo-image-elements.git

1. Ensure dependencies are installed and required virsh configuration is performed:

        cd $TRIPLEO_ROOT/incubator
        scripts/install-dependencies

1. Configure a network for your test environment.
   This configures libvirt to setup a bridge with no ip address and adds an
   exclusion for dnsmasq so it does not listen on your test network. Note that
   the order of the parameters to bm_poseur is significant : copy-paste this
   line.

        cd $TRIPLEO_ROOT/bm_poseur/
        sudo ./bm_poseur --bridge-ip=none create-bridge

1. Create and start your seed VM. This script invokes diskimage-builder with
   suitable paths and options to create and start a VM that contains an
   all-in-one OpenStack cloud with the baremetal driver enabled, and
   preconfigures it for a development environment.

        cd $TRIPLEO_ROOT/tripleo-image-elements/elements/boot-stack
        sed -i "s/\"user\": \"stack\",/\"user\": \"`whoami`\",/" config.json
        cd $TRIPLEO_ROOT/incubator/

    To build an Ubuntu node:

        scripts/boot-elements boot-stack ubuntu cloud-init-nocloud -o seed

    To build a Fedora node:

        scripts/boot-elements boot-stack fedora disable-selinux -o seed


    Your SSH pub key has been copied to the resulting 'seed' VMs root
    user.  It has been started by the boot-elements script, and can be logged
    into at this point.

1. Get the IP address and hostname of the VM

        SEED_IP=`scripts/get-vm-ip seed`
        SEED_HOSTNAME=`scripts/get-vm-hostname seed`

1. Mask the SEED_IP out of your proxy settings

        export no_proxy=$no_proxy,$SEED_IP

1. If you downloaded a pre-built seed image you will need to log into it
   and customise the configuration with in it. See footnote [1].)

1. Create some 'baremetal' node(s) out of KVM virtual machines.
   Nova will PXE boot these VMs as though they were physical hardware.
   You can use bm_poseur to automate this, or if you want to create
   the VMs yourself, see footnote [2] for details on their requirements.

        sudo $TRIPLEO_ROOT/bm_poseur/bm_poseur --vms 3 --arch i686 create-vm

    __(Note: if you have set http_proxy or https_proxy to a network host, you must either configure that network host to route traffic to your VM ip properly, or add the SEED_IP to your no_proxy environment variable value.)__

1. Nova tools have been installed in $TRIPLEO_ROOT/incubator/scripts - you need
   to add that to the PATH (unless you have them installed already).

        export PATH=$PATH:$TRIPLEO_ROOT/incubator/scripts

1. Copy the openstack credentials out of the seed VM, and add the IP:
   (https://bugs.launchpad.net/tripleo/+bug/1191650)

        scp root@$SEED_IP:stackrc $TRIPLEO_ROOT/seedrc
        sed -i "s/localhost/$SEED_IP/" $TRIPLEO_ROOT/seedrc
        source $TRIPLEO_ROOT/seedrc

1. Get the list of MAC addresses for all the VMs you have created.

        export MACS=`$TRIPLEO_ROOT/bm_poseur/bm_poseur get-macs`

1. Perform setup of your cloud. The 1 512 10 is CPU count, memory in MB, disk
   in GB for your test nodes.
   XXX: need to only use the first node (the seed)

        user-config
        setup-baremetal $SEED_HOSTNAME 1 512 10

1. Create your undercloud image. This is the image that the seed nova
   will deploy to become the baremetal undercloud.

        export ELEMENTS_PATH=$TRIPLEO_ROOT/tripleo-image-elements/elements

    To build an Ubuntu undercloud node:

        $TRIPLEO_ROOT/diskimage-builder/bin/disk-image-create -u -a i386 -o undercloud boot-stack ubuntu

    To build a Fedora undercloud node:

        $TRIPLEO_ROOT/diskimage-builder/bin/disk-image-create -u -a i386 -o undercloud boot-stack fedora disable-selinux

1. Load the undercloud image into Glance:

        $TRIPLEO_ROOT/incubator/scripts/load-image undercloud.qcow2

1. Allow the VirtualPowerManager to ssh into your host machine to power on vms:

        ssh root@$SEED_IP "cat /opt/stack/boot-stack/virtual-power-key.pub" >> ~/.ssh/authorized_keys

1. Start the process of provisioning a baremetal node:
   XX: This should be 'heat stack-create'.

        nova boot --flavor baremetal --image undercloud --key_name default bmtest

   You can watch its console to observe the PXE boot/deploy process.
   After the deploy is complete, it will reboot into the image.

1. Add a route to the baremetal bridge via the seed node (we do this so that
   your host is isolated from the networking of the test environment.

        ip route add 192.0.2.0/24 dev virbr0 via $SEED_IP

1. Get the undercloud IP from 'nova list'

1. Copy the stackrc out of the undercloud:

        scp root@$UNDERCLOUD_IP:stackrc $TRIPLEO_ROOT/undercloudrc
        sed -i "s/localhost/$UNDERCLOUD_IP/" $TRIPLEO_ROOT/undercloudrc
        source $TRIPLEO_ROOT/undercloudrc

1. Perform setup of your undercloud. The 1 512 10 is CPU count, memory in MB, disk
   in GB for your test nodes.
   XXX: need to mask out the first node (the seed)

        user-config
        setup-baremetal $SEED_HOSTNAME 1 512 10

The End!



Footnotes
=========

* [1] Customize a downloaded seed image.

  If you downloaded your seed VM image, you may need to configure it.
  Setup a network proxy, if you have one (e.g. 192.168.2.1 port 8080)

        echo << EOF >> ~/.profile
        export no_proxy=192.0.2.1
        export http_proxy=http://192.168.2.1:8080/
        EOF

  Add an ~/.ssh/authorized_keys file. The image rejects password authentication
  for security, so you will need to ssh out from the VM console. Even if you
  don't copy your authorized_keys in, you will still need to ensure that
  /home/stack/.ssh/authorized_keys on your seed node has some kind of
  public SSH key in it, or the openstack configuration scripts will error.

* [2] Requirements for the "baremetal node" VMs

  If you don't use bm_poseur, but want to create your own VMs, here are some
  suggestions for what they should look like.
   - each VM should have two NICs
   - both NICs should be on br99
   - record the MAC addresses for each NIC
   - give each VM no less than 2GB of disk, and ideally give them
     more than BM_FLAVOR_ROOT_DISK, which defaults to 2GB
   - 512MB RAM is probably enough
   - if using KVM, specify that you will install the virtual machine via PXE.
     This will avoid KVM prompting for a disk image or installation media.

* [3] Setting Up Squid Proxy

  - Install squid proxy: `apt-get install squid`
  - Set `/etc/squid3/squid.conf` to the following:
<pre><code>
          acl manager proto cache_object
          acl localhost src 127.0.0.1/32 ::1
          acl to_localhost dst 127.0.0.0/8 0.0.0.0/32 ::1
          acl localnet src 10.0.0.0/8 # RFC1918 possible internal network
          acl localnet src 172.16.0.0/12  # RFC1918 possible internal network
          acl localnet src 192.168.0.0/16 # RFC1918 possible internal network
          acl SSL_ports port 443
          acl Safe_ports port 80      # http
          acl Safe_ports port 21      # ftp
          acl Safe_ports port 443     # https
          acl Safe_ports port 70      # gopher
          acl Safe_ports port 210     # wais
          acl Safe_ports port 1025-65535  # unregistered ports
          acl Safe_ports port 280     # http-mgmt
          acl Safe_ports port 488     # gss-http
          acl Safe_ports port 591     # filemaker
          acl Safe_ports port 777     # multiling http
          acl CONNECT method CONNECT
          http_access allow manager localhost
          http_access deny manager
          http_access deny !Safe_ports
          http_access deny CONNECT !SSL_ports
          http_access allow localnet
          http_access allow localhost
          http_access deny all
          http_port 3128
          cache_dir aufs /var/spool/squid3 5000 24 256
          maximum_object_size 1024 MB
          coredump_dir /var/spool/squid3
          refresh_pattern ^ftp:       1440    20% 10080
          refresh_pattern ^gopher:    1440    0%  1440
          refresh_pattern -i (/cgi-bin/|\?) 0 0%  0
          refresh_pattern (Release|Packages(.gz)*)$      0       20%     2880
          refresh_pattern .       0   20% 4320
          refresh_all_ims on
         </pre></code>

 - Restart squid: `sudo service squid3 restart`
 - Set http_proxy environment variable: `http_proxy=http://your_ip_or_localhost:3128/`

