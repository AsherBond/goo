
[Howto: Using UFO (swift) — A Quick Setup Guide]
------------------------------------------------

This sets up a GlusterFS Unified File and Object (UFO) server on a
single node (single brick) Gluster server using the RPMs contained in my
YUM repo at
[http://repos.fedorapeople.org/repos/kkeithle/glusterfs/](http://repos.fedorapeople.org/repos/kkeithle/glusterfs/).
This repo contains RPMs for Fedora 16, Fedora 17, and RHEL 6.

​1. Add the repo to your system. See the README file there for
instructions. N.B. If you’re using CentOS or some other RHEL clone
you’ll want (need) to add the Fedora EPEL repo — see
[http://fedoraproject.org/wiki/EPEL](http://fedoraproject.org/wiki/EPEL).

​2. Install glusterfs and UFO (remember to enable the new repo first):

`` `yum install glusterfs glusterfs-server glusterfs-fuse glusterfs-swift glusterfs-swift-account glusterfs-swift-container glusterfs-swift-object glusterfs-swift-proxy glusterfs-ufo` ``

​3. Start glusterfs:

-   On Fedora 17, Fedora 18: `` `systemctl start glusterd.service` ``
-   On Fedora 16 or  RHEL 6 `` `service start glusterd` ``
-   ``On CentOS6.x `` `/etc/init.d/glusterd start` ``

​4. Create a glusterfs volume:\
 `` `gluster volume create $myvolname $myhostname:$pathtobrick` ``

​5. Start the glusterfs volume:\
 `` `gluster volume start $myvolname` ``

​6. Create a self-signed cert for UFO:\

`` `cd /etc/swift; openssl req -new -x509 -nodes -out cert.crt -keyout cert.key` ``

​7. fixup some files in /etc/swift:

-   `` `mv swift.conf-gluster swift.conf` ``
-   `` `mv fs.conf-gluster fs.conf` ``
-   `` `mv proxy-server.conf-gluster proxy-server.conf` ``
-   `` `mv account-server/1.conf-gluster account-server/1.conf` ``
-   `` `mv container-server/1.conf-gluster container-server/1.conf` ``
-   `` `mv object-server/1.conf-gluster object-server/1.conf` ``
-   `` `rm {account,container,object}-server.conf ``

​8. Configure UFO (edit /etc/swift/proxy-server.conf):\
 + add your cert and key to the [DEFAULT] section:\
 bind\_port = 443\
 cert\_file = /etc/swift/cert.crt\
 key\_file = /etc/swift/cert.key\
 + add one or more users of the gluster volume to the [filter:tempauth]
section:\
 user\_\$myvolname\_\$username=\$password .admin\
 + add the memcache address to the [filter:cache] section:\
 memcache\_servers = 127.0.0.1:11211

​9. Generate builders:\
 `` `/usr/bin/gluster-swift-gen-builders $myvolname` ``

​10. Start memcached:

-   On Fedora 17: `` `systemctl start memcached.service` ``
-   On Fedora 16 or  RHEL 6 `` `service start memcached` ``
-   ``On CentOS6.x `` `/etc/init.d/memcached start` ``

​11. Start UFO:

`` `swift-init main start` ``

» This has bitten me more than once. If you ssh -X into the machine
running swift, it’s likely that sshd will already be using ports 6010,
6011, and 6012, and will collide with the swift processes trying to use
those ports «

​12. Get authentication token from UFO:\

`` `curl -v -H 'X-Storage-User: $myvolname:$username' -H 'X-Storage-Pass: $password' -k https://$myhostname:443/auth/v1.0` ``\
 (authtoken similar to AUTH\_tk2c69b572dd544383b352d0f0d61c2e6d)

​13. Create a container:\

`` `curl -v -X PUT -H 'X-Auth-Token: $authtoken' https://$myhostname:443/v1/AUTH_$myvolname/$mycontainername -k` ``

​14. List containers:\

`` `curl -v -X GET -H 'X-Auth-Token: $authtoken' https://$myhostname:443/v1/AUTH_$myvolname -k` ``

​15. Upload a file to a container:

`` `curl -v -X PUT -T $filename -H 'X-Auth-Token: $authtoken' -H 'Content-Length: $filelen' https://$myhostname:443/v1/AUTH_$myvolname/$mycontainername/$filename -k` ``

​16. Download the file:

`` `curl -v -X GET -H 'X-Auth-Token: $authtoken' https://$myhostname:443/v1/AUTH_$myvolname/$mycontainername/$filename -k > $filename` ``

More information and examples are available from

-   [http://docs.openstack.org/api/openstack-object-storage/1.0/content/](http://docs.openstack.org/api/openstack-object-storage/1.0/content/)
-   [https://access.redhat.com/knowledge/docs/en-US/Red\_Hat\_Storage/2.0/pdf/Administration\_Guide/Red\_Hat\_Storage-2.0-Administration\_Guide-en-US.pdf](https://access.redhat.com/knowledge/docs/en-US/Red_Hat_Storage/2.0/pdf/Administration_Guide/Red_Hat_Storage-2.0-Administration_Guide-en-US.pdf)
-   [http://docs.openstack.org/developer/swift/howto\_installmultinode.html](http://docs.openstack.org/developer/swift/howto_installmultinode.html)

=======================================================================

N.B. We (Red Hat, Gluster) generally recommend using xfs for brick
volumes; or if you’re feeling brave, btrfs. If you’re using ext4 be
aware of the ext4 issue\* and if you’re using ext3 make sure you mount
it with -o user\_xattr.

\* http://joejulian.name/blog/glusterfs-bit-by-ext4-structure-change/

