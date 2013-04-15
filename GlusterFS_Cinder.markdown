
With EL6, configuring OpenStack Grizzly to use Gluster for its Cinder
(block) storage is fairly simple.

These instructions have been tested with both GlusterFS 3.3 and
GlusterFS 3.4. Other releases may also work, but have not been tested.

Before beginning, it is very important to ensure there are **no existing
volumes** in Cinder. Use "cinder delete" to remove any, and "cinder
list" to verify they are gone. If you don’t, it will cause errors later
in the process, breaking your Cinder installation.

**NOTE** - Unlike other software, the "openstack-config" and "cinder"
commands generally require you to run them as root. Without prior
configuration, running them through sudo generally doesn’t work. (this
can be changed, but is beyond the scope of this HOW-TO)

    +

1. Install GlusterFS client
===========================

On each Cinder host, install the GlusterFS client packages:

       $ sudo yum -y install glusterfs-fuse

    +

2. Add GlusterFS to Cinder configuration
========================================

On each Cinder host, run these commands to add GlusterFS to the Cinder
configuration:

       # openstack-config --set cinder.conf DEFAULT volume_driver cinder.volume.drivers.glusterfs.GlusterfsDriver
       # openstack-config --set cinder.conf DEFAULT glusterfs_shares_config /etc/cinder/shares.conf
       # openstack-config --set cinder.conf DEFAULT glusterfs_mount_point_base /var/lib/cinder/volumes

    +

3. Create GlusterFS volume list
===============================

On each of the Cinder nodes, create the simple text file
**/etc/cinder/shares.conf**.

This file is a simple list of the GlusterFS volumes to be used, one per
line, in the format:

       GLUSTERHOST:VOLUME
       GLUSTERHOST:NEXTVOLUME
       GLUSTERHOST2:SOMEOTHERVOLUME

For example:

       myglusterbox.example.org:myglustervol

    +

4. Update the glusterfs.py driver file (if needed)
==================================================

If the python-cinder package installed on your system is 2013.1-0.5.rc3
or earlier, its glusterfs.py file is incompatible with Grizzly.

A Grizzly compatible version is being worked on, but might take a week
or two to be ready.

In the meantime, download the temporary replacement file from
[here](https://github.com/justinclift/miscellaneous/raw/master/glusterfs/glusterfs.py),
and use it to overwrite this file:

       /usr/lib/python2.6/site-packages/cinder/volume/drivers/glusterfs.py

Ensure the ownership and permissions of the file are acceptable in your
environment:

       $ sudo chown root:root /usr/lib/python2.6/site-packages/cinder/volume/drivers/glusterfs.py
       $ sudo chmod 644 /usr/lib/python2.6/site-packages/cinder/volume/drivers/glusterfs.py

    +

5. Update firewall for GlusterFS
================================

You’ll also need to update the firewall rules on each Cinder node, so
they can communicate with the Gluster nodes.

The ports to open are explained here (under item 3):

       http://gluster.org/community/documentation/index.php/Gluster_3.2:_Installing_GlusterFS_on_Red_Hat_Package_Manager_(RPM)_Distributions

If you’re using iptables as your firewall, these lines can be added
under **:OUTPUT ACCEPT** in the "\*filter" section. You should probably
adjust them to suit your environment (eg. only accept connections from
your GlusterFS servers).

       -A INPUT -m state --state NEW -m tcp -p tcp --dport 111 -j ACCEPT
       -A INPUT -m state --state NEW -m tcp -p tcp --dport 24007 -j ACCEPT
       -A INPUT -m state --state NEW -m tcp -p tcp --dport 24008 -j ACCEPT
       -A INPUT -m state --state NEW -m tcp -p tcp --dport 24009 -j ACCEPT
       -A INPUT -m state --state NEW -m tcp -p tcp --dport 24010 -j ACCEPT
       -A INPUT -m state --state NEW -m tcp -p tcp --dport 24011 -j ACCEPT
       -A INPUT -m state --state NEW -m tcp -p tcp --dport 38465:38469 -j ACCEPT

Restart the firewall service:

       $ sudo service iptables restart

    +

6. Restart Cinder volume service
================================

Configuration is complete, so restart the Cinder volume service to make
it active.

       $ sudo service openstack-cinder-volume restart

It’s a good idea to check Cinder volume log after doing so, to make sure
there are no errors:

       $ sudo tail -50 /var/log/cinder/volume.log

    +

7. Verify GlusterFS is being used
=================================

To make sure everything is working as desired, create a Cinder volume
then check its using GlusterFS.

Create a Cinder volume:

       # cinder create --display_name myvol 10

Wait a few seconds for the creation, then do:

       # cinder list

The volume should be in "available" status. Now look for a new file in
the GlusterFS volume directory:

       $ sudo ls -lah /var/lib/cinder/volumes/XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX/

(the XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX will be a number specific to your
installation)

A newly created file should be inside that directory. That’s the new
volume you just created. A new file will appear each time you create a
volume.

For example:

       $ sudo ls -lah /var/lib/cinder/volumes/29e55f0f3d56494ef1b1073ab927d425/
       total 4.0K
       drwxr-xr-x. 3 root   root     73 Apr  4 15:46 .
       drwxr-xr-x. 3 cinder cinder 4.0K Apr  3 09:31 ..
       -rw-rw-rw-. 1 root   root    10G Apr  4 15:46 volume-a4b97d2e-0f8e-45b2-9b94-b8fa36bd51b9

With that, it’s all done. :)
