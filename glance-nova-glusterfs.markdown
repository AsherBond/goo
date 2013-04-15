% OpenStack Glance And Nova Backed By Red Hat Storage | Tropical Devel
% 
% 

[Tropical Devel](https://tropicaldevel.wordpress.com/)
======================================================

Bits of code from the pacific 

[](https://tropicaldevel.wordpress.com/)

-   [About](https://tropicaldevel.wordpress.com/about-2/)

Posted by: **John Bresnahan, Red Hat** | January 16, 2013

OpenStack Glance And Nova Backed By Red Hat Storage
---------------------------------------------------

In this post I will explain how to configure Glance and Nova to be
backed by Red Hat Storage (and thereby gluster).

The ISOs that I link to in this post require a Red Hat Network
subscription.  The information here can be useful without it, but it is
less convenient.

Deployment Details
------------------

In my deployment I use two virtual machines.  One for Red Hat Storage
and the other for all of the OpenStack services (Keystone, Glance, and
Nova).  Both VMs run on my Lenovo T530 laptop which runs Fedora 17.  I
will not go into detail about how to create VMs because that is covered
pretty well in other places, but I will touch on it just to make it as
clear as possible.

Red Hat Storage VM
------------------

For the base VM I wanted a lot of space so I created a VM with 48GB of
storage (10 should be plenty) and configured it to have 1GB of RAM and 1
CPU.

1.  Get Red Hat Storage by going
    [here](https://rhn.redhat.com/rhn/software/channel/downloads/Download.do?cid=14689)
    and clicking on *Binary DVD*.
2.  Create a VM with at least 8 GB of storage (mine had 48) and install
    Red Hat Storage by following the instructions
    [here](https://access.redhat.com/knowledge/docs/en-US/Red_Hat_Storage/2.0/html/Installation_Guide/chap-Installation_Guide-Install_RHS.html#sect-Installation_Guide-Install_RHS-install_frm_iso_1)
    through step 7.1.
3.  Create a gluster volume with the following commands:

~~~~
{.brush: .bash; .collapse: .false; .title: .; .wrap-lines: .false; .notranslate title=""}
gluster volume create testvol myhost:/exp1
gluster volume start testvol
/etc/init.d/glusterd restart
~~~~

The VM should now be ready to serve a gluster file system.

OpenStack VM
------------

Here we will use a single VM for all of the OpenStack services.  The
explanation of how to back either Glance or Nova by gluster will be the
same for a distributed environment.

Create a VM with 8GB of storage using RHEL 6.4 (64 bit) by downloading
the binary DVD available
[here](https://rhn.redhat.com/rhn/software/channel/downloads/Download.do?cid=10156)

Install OpenStack by following the instructions
[here](https://access.redhat.com/knowledge/docs/en-US/Red_Hat_OpenStack_Preview/2/html/Getting_Started_Guide/chapter-Nova.html)
through step 6.

At this point you should have a VM which is running enough OpenStack
services to upload VMs, launch them, and ssh into them.  The
instructions referenced above should provide you with steps to verify
this.

note: in selinux I had to run the following command in order to properly
boot images:

~~~~
{.brush: .bash; .collapse: .false; .title: .; .wrap-lines: .false; .notranslate title=""}
setenforce permissive
~~~~

You may also need to open up /etc/nova/nova.conf and make sure you have
the line:

~~~~
{.brush: .bash; .collapse: .false; .title: .; .wrap-lines: .false; .notranslate title=""}
libvirt_type = qemu
~~~~

Mount Gluster
-------------

The OpenStack VM now needs to mount the storage system provided by Red
Hat Storage so that it can be used by Glance and Nova.  First the VM
must be configured so that the recommend gluster client can be used. 
The instructions for this are
[here](https://access.redhat.com/knowledge/articles/145553), but I will
put them in my own words.

Goto
[https://access.redhat.com/subscriptions/rhntransition](https://access.redhat.com/subscriptions/rhntransition/)
and navigate: Subscriptions -\> RHN Classic -\> Registered Systems.

Find your system and click on it.  Click on ‘Alter Channel
Subscriptions” on the new page.  Find *Additional Services Channels for
Red Hat Enterprise Linux 6 for x86\_64*  and expand it.  Select*Red Hat
Storage Native Client (RHEL Server 6 for x86\_64)*then click the *Update
Subscription* button.

Now on the OpenStack machine run:

~~~~
{.brush: .bash; .collapse: .false; .title: .; .wrap-lines: .false; .notranslate title=""}
yum install glusterfs-fuse glusterfs
mkdir -p /mnt/gluster/
mount -t glusterfs <storage VM IP>:/testvol /mnt/gluster
~~~~

At this point the OpenStack VM should be able to access the gluster file
system.

Configure Glance
----------------

In order to change the path that Glance uses for its file system store
only a single line in /etc/glance/glance-api.conf needs to be changed.

~~~~
{.brush: .bash; .collapse: .false; .title: .; .wrap-lines: .false; .notranslate title=""}
filesystem_store_datadir = /mnt/gluster/glance/images
~~~~

Now run the following commands

~~~~
{.brush: .bash; .collapse: .false; .title: .; .wrap-lines: .false; .notranslate title=""}
mkdir -p /mnt/gluster/glance/images
chown -R glance:glance /mnt/gluster/glance/
create the directory for the instance store
mkdir /mnt/gluster/instance/
chown -R nova:nova  /mnt/gluster/instance/
service openstack-glance-api restart
~~~~

At this point glance should be backed by the gluster file system.  Lets
upload a file to glance and verify this.  A good test image is available
[here](http://berrange.fedorapeople.org/images/2012-02-29/f16-x86_64-openstack-sda.qcow2).

~~~~
{.brush: .bash; .collapse: .false; .title: .; .wrap-lines: .false; .notranslate title=""}
glance image-create --name="test" --is-public=true --container-format=ovf --disk-format=qcow2 < f16-x86_64-openstack-sda.qcow2
ls -l /mnt/gluster/glance/images
~~~~

Configure Nova
--------------

The final step is to configure Nova such that nova-compute uses gluster
for its instance store.  The instance store is the temporary area to
which the VM is copied and then booted.  Just as it was with Glance,
configuring nova to use gluster in this way is a simple one line file
change.  Open the file /etc/nova/nova.conf and file the key
*instances\_path*.  Change the line to be the following:

~~~~
{.brush: .bash; .collapse: .false; .title: .; .wrap-lines: .false; .notranslate title=""}
instances_path = /mnt/gluster/instance
~~~~

Now setup the correct paths and permissions and restart nova-compute.

~~~~
{.brush: .bash; .collapse: .false; .title: .; .wrap-lines: .false; .notranslate title=""}
mkdir  -p /mnt/gluster/instance/
chown -R nova:nova  /mnt/gluster/instance/
service openstack-nova-compute restart
~~~~

That should be all that is needed.

Future Work
-----------

The idea behind this is that if Glance and Nova are backed by the same
file system that image propagation should be much faster.  In the future
I will be looking for a testbed where I can verify this.

Update
------

You should now be able to automatically mount on boot, add the following
to you /etc/fstab file:

~~~~
{.brush: .bash; .collapse: .false; .title: .; .wrap-lines: .false; .notranslate title=""}
<gluster IP>:/glustervol /mnt/gluster glusterfs defaults,_netdev 0 0
~~~~


