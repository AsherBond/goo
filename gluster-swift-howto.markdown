% Howto: Using UFO (swift) — A Quick Setup Guide | Gluster Community
  Website
% 
% 

[![Gluster.Org](http://www.gluster.org/wp-content/themes/gluster1/images/logo.png)](http://www.gluster.org)

[Tweet](http://twitter.com/share)

This project sponsored by [Red Hat, Inc.](http://www.redhat.com/) ›

-   [![Documentation](http://www.gluster.org/wp-content/themes/gluster1/images/documentation.png)](http://www.gluster.org/community/documentation/index.php/Main_Page)
-   [![Contact](http://www.gluster.org/wp-content/themes/gluster1/images/contact.png)](http://www.gluster.org/interact)
-   [![About](http://www.gluster.org/wp-content/themes/gluster1/images/about.png)](http://www.gluster.org/about/)
-   [![Developers](http://www.gluster.org/wp-content/themes/gluster1/images/developers.png)](http://www.gluster.org/community/documentation/index.php/Developers)
-   [![Download](http://www.gluster.org/wp-content/themes/gluster1/images/download.png)](http://www.gluster.org/download)

by
[kkeithley](http://www.gluster.org/author/kkeithley/ "Posts by kkeithley")
on September 17, 2012

[← Return to Blog Home](/blog)

[Howto: Using UFO (swift) — A Quick Setup Guide](http://www.gluster.org/2012/09/howto-using-ufo-swift-a-quick-and-dirty-setup-guide/)
-------------------------------------------------------------------------------------------------------------------------------------

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

[Tweet](http://twitter.com/share)

### 20 Comments {#comments-title}

1.  [A Buzzword-Packed Return to Gluster UFO |
    jebpages](http://blog.jebpages.com/archives/a-buzzword-packed-return-to-gluster-ufo/)
    says:

    [October 11, 2012 at 9:52
    am](http://www.gluster.org/2012/09/howto-using-ufo-swift-a-quick-and-dirty-setup-guide/comment-page-1/#comment-2638)  [(Edit)](http://www.gluster.org/wp-admin/comment.php?action=editcomment&c=2638 "Edit comment")

    [...] little while back, I tested out the Unified File and Object
    feature in Gluster 3.3, which taps OpenStack’s Swift component to
    handle the object half of the file and object combo. It took me kind
    of a long time to [...]

2.  Rob says:

    [December 16, 2012 at 2:50
    am](http://www.gluster.org/2012/09/howto-using-ufo-swift-a-quick-and-dirty-setup-guide/comment-page-1/#comment-19309)  [(Edit)](http://www.gluster.org/wp-admin/comment.php?action=editcomment&c=19309 "Edit comment")

    Thanks for the howto document! I’m looking to test this out for a
    project I’m working on, so this was very helpful. One thing though…
    when I try to perform step 7, I see that /etc/swift just contains a
    few empty directories, and not the files you have indicated. Does
    this imply that I forgot to install a package? Thanks!

3.  kkeithley says:

    [December 17, 2012 at 6:02
    am](http://www.gluster.org/2012/09/howto-using-ufo-swift-a-quick-and-dirty-setup-guide/comment-page-1/#comment-19353)  [(Edit)](http://www.gluster.org/wp-admin/comment.php?action=editcomment&c=19353 "Edit comment")

    I missed an edit when I updated the post. Install the
    glusterfs-swift-ufo rpm — instead of glusterfs-swift-plugin rpm —
    and you’ll get the files you need.

4.  Rob says:

    [December 17, 2012 at 11:34
    am](http://www.gluster.org/2012/09/howto-using-ufo-swift-a-quick-and-dirty-setup-guide/comment-page-1/#comment-19358)  [(Edit)](http://www.gluster.org/wp-admin/comment.php?action=editcomment&c=19358 "Edit comment")

    That fixed it… thanks! Once again, great howto. Thanks for putting
    it together.

5.  Rob says:

    [December 17, 2012 at 5:04
    pm](http://www.gluster.org/2012/09/howto-using-ufo-swift-a-quick-and-dirty-setup-guide/comment-page-1/#comment-19370)  [(Edit)](http://www.gluster.org/wp-admin/comment.php?action=editcomment&c=19370 "Edit comment")

    Hmm… new issue. Everything seems to work great, except for
    downloading files. No matter how I try to do it (curl, swift, etc) I
    wind up with a 503 Internal Server Error. The errors printed in
    /var/log/messages are too long to list here, but here’s a small
    sample:

    Dec 17 10:00:37 37 object-server ERROR \_\_call\_\_ error with GET
    /bank0/0/AUTH\_bank0/testcontainer/test\_file : \#012Traceback (most
    recent call last):\#012 File
    “/usr/lib/python2.6/site-packages/swift/obj/server.py”, line 892, in
    \_\_call\_\_\#012 res = method(req)\#012 File
    “/usr/lib/python2.6/site-packages/swift/common/utils.py”, line 1350,
    in wrapped\#012 return func(\*a, \*\*kw)\#012 File
    “/usr/lib/python2.6/site-packages/swift/obj/server.py”, line 677, in
    GET\#012 iter\_hook=sleep)\#012TypeError: \_\_init\_\_() got an
    unexpected keyword argument ‘iter\_hook’ (txn:
    tx47b9a96c72424f1889bbeb976d185498)\
     Dec 17 10:00:37 37 object-server 127.0.0.1 – -
    [17/Dec/2012:18:00:37 +0000] “GET
    /bank0/0/AUTH\_bank0/testcontainer/test\_file” 500 425 “-”
    “tx47b9a96c72424f1889bbeb976d185498″ “-” 0.0013\
     Dec 17 10:00:37 37 proxy-server ERROR 500 Traceback (most recent
    call last):\#012 File
    “/usr/lib/python2.6/site-packages/swift/obj/server.py”, line 892, in
    \_\_call\_\_\#012 res = method(req)\#012 File
    “/usr/lib/python2.6/site-packages/swift/common/utils.py”, line 1350,
    in wrapped\#012 return func(\*a, \*\*kw)\#012 File
    “/usr/lib/python2.6/site-packages/swift/obj/server.py”, line 677, in
    GET\#012 iter\_hook=sleep)\#012TypeError: \_\_init\_\_() got an
    unexpected keyword argument ‘iter\_hook’\#012 From Object Server
    127.0.0.1:6010 (txn: tx47b9a96c72424f1889bbeb976d185498)
    (client\_ip: 172.16.1.9)\
     Dec 17 10:00:37 37 proxy-server Object GET returning 503 for [500]
    (txn: tx47b9a96c72424f1889bbeb976d185498) (client\_ip: 172.16.1.9)

    Any ideas? Everything else works so perfectly. And by the way, the
    swift client and the glusterfs server are on the same private vlan,
    no firewall between them.

    And finally, the command I executed to download:

    swift -A
    [https://\$hostname:443/auth/v1.0](https://$hostname:443/auth/v1.0)
    -U \$volume:\$user -K \$password download \$container \$file

6.  kkeithley says:

    [December 21, 2012 at 11:31
    am](http://www.gluster.org/2012/09/howto-using-ufo-swift-a-quick-and-dirty-setup-guide/comment-page-1/#comment-19534)  [(Edit)](http://www.gluster.org/wp-admin/comment.php?action=editcomment&c=19534 "Edit comment")

    New RPMs are available in my fedorapeople.org yum repo to fix the
    bug.

    Note that one of the RPMs has been renamed: glusterfs-swift-plugin
    -\> glusterfs-swift-ufo -\> glusterfs-ufo. I found that an
    orderinary update didn’t work and I had to first remove, then add.
    All your volume files and swift configuration will be retained.

7.  Rob says:

    [December 26, 2012 at 3:42
    pm](http://www.gluster.org/2012/09/howto-using-ufo-swift-a-quick-and-dirty-setup-guide/comment-page-1/#comment-19763)  [(Edit)](http://www.gluster.org/wp-admin/comment.php?action=editcomment&c=19763 "Edit comment")

    I got the correct packages installed, and that did fix the errors I
    listed above. But now, all downloads stall after downloading the
    first 65536 (2\^16) bytes. I’ve only tested with curl and the swift
    client. But I don’t see anything in any configurations that would
    impose this limit. Any ideas?

8.  Rob says:

    [January 3, 2013 at 4:34
    pm](http://www.gluster.org/2012/09/howto-using-ufo-swift-a-quick-and-dirty-setup-guide/comment-page-1/#comment-20415)  [(Edit)](http://www.gluster.org/wp-admin/comment.php?action=editcomment&c=20415 "Edit comment")

    Is anyone else using this and noticing that downloads stall after
    the first 64K bytes (65536 to be exact)?

    I’ve tried tweaking the settings to avoid this. For example, setting
    ‘object\_chunk\_size’ and ‘client\_chunk\_size’ in proxy-server.conf
    to larger sizes, but that doesn’t help. I’ve set them as large as
    1GB.

    The only way I can successfully download anything larger than 64k is
    if I first upload it via the swift client and use ‘-S 65536′ to make
    sure it breaks the file up into 64K segments or smaller.

9.  kkeithley says:

    [January 4, 2013 at 4:46
    am](http://www.gluster.org/2012/09/howto-using-ufo-swift-a-quick-and-dirty-setup-guide/comment-page-1/#comment-20451)  [(Edit)](http://www.gluster.org/wp-admin/comment.php?action=editcomment&c=20451 "Edit comment")

    We’re working on a fix for the download stall.

10. Rob says:

    [January 8, 2013 at 1:08
    pm](http://www.gluster.org/2012/09/howto-using-ufo-swift-a-quick-and-dirty-setup-guide/comment-page-1/#comment-20827)  [(Edit)](http://www.gluster.org/wp-admin/comment.php?action=editcomment&c=20827 "Edit comment")

    That’s great news! I’ll look forward to that.

11. kkeithley says:

    [January 10, 2013 at 12:00
    pm](http://www.gluster.org/2012/09/howto-using-ufo-swift-a-quick-and-dirty-setup-guide/comment-page-1/#comment-21095)  [(Edit)](http://www.gluster.org/wp-admin/comment.php?action=editcomment&c=21095 "Edit comment")

    I’ve just put up new RPMs with the fix in my fedorapeople.org yum
    repository.

    3.3.1-8, for fedora 16, fedora 17, epel 5, epel 6, and rhel 7 beta.

    RPMs for fedora 18 will appear in the updates yum repo after the
    requisite testing period, after f18 ships in a few days.

12. Rob says:

    [January 11, 2013 at 5:54
    pm](http://www.gluster.org/2012/09/howto-using-ufo-swift-a-quick-and-dirty-setup-guide/comment-page-1/#comment-21281)  [(Edit)](http://www.gluster.org/wp-admin/comment.php?action=editcomment&c=21281 "Edit comment")

    I installed the updates, and so far, so good! I’m only playing
    around with file sizes less than 2GB. But when I get some time, I’ll
    be pushing larger stuff (\~200GB ProRes video files). I imagine that
    will require some config tweaks to allow to work, and to deal with
    Swift’s 5GB limits.

    Thanks for the help once again!

13. Kaleb says:

    [January 14, 2013 at 4:44
    am](http://www.gluster.org/2012/09/howto-using-ufo-swift-a-quick-and-dirty-setup-guide/comment-page-1/#comment-21524)  [(Edit)](http://www.gluster.org/wp-admin/comment.php?action=editcomment&c=21524 "Edit comment")

    UFO ships with the 5GB limit already disabled.

14. Rob says:

    [January 17, 2013 at 1:51
    pm](http://www.gluster.org/2012/09/howto-using-ufo-swift-a-quick-and-dirty-setup-guide/comment-page-1/#comment-22064)  [(Edit)](http://www.gluster.org/wp-admin/comment.php?action=editcomment&c=22064 "Edit comment")

    I can confirm uploads/downloads of 200GB+ with no issues. It’s
    stressing the NICs on my servers, but that isn’t Swift’s fault!
    Thanks for the guide, and especially thanks for rapidly responding
    with updates to the packages, etc.

15. Sachin says:

    [January 23, 2013 at 11:26
    am](http://www.gluster.org/2012/09/howto-using-ufo-swift-a-quick-and-dirty-setup-guide/comment-page-1/#comment-23178)  [(Edit)](http://www.gluster.org/wp-admin/comment.php?action=editcomment&c=23178 "Edit comment")

    Hi there,

    Thanks for the guide. Before I start, I wanted to learn of there is
    any performance/benchmark study on this way of doing the Object
    store. Reason to ask is that Openstack Swift is really sub-optimal
    in RAID back-ends (since it does may small writes) as noted here

    [http://docs.openstack.org/trunk/openstack-object-storage/admin/content/raid.html](http://docs.openstack.org/trunk/openstack-object-storage/admin/content/raid.html)

    On the other hand, any sizable Gluster deployment probably uses RAID
    in a big way – irc has several conversations around how to setup a
    big Gluster deployment (and this uses RAID). See for example

    [http://irclog.perlgeek.de/gluster/2012-12-01](http://irclog.perlgeek.de/gluster/2012-12-01)

    So, any benchmarks anyone has with UFO?

    Thanks in advance!

    -Sachin

16. Rob says:

    [January 28, 2013 at 7:13
    pm](http://www.gluster.org/2012/09/howto-using-ufo-swift-a-quick-and-dirty-setup-guide/comment-page-1/#comment-24631)  [(Edit)](http://www.gluster.org/wp-admin/comment.php?action=editcomment&c=24631 "Edit comment")

    @sachin – I haven’t done any really scientific benchmarking, but I
    did some throughput tests to ensure the content I am storing on
    GlusterFS volumes is accessible with a reasonable degree of
    performance. In my tests, I was moving files ranging in size between
    30GB and 200GB, and in all cases, with UFO, I was seeing
    \~120-130Mbps tranfser rates… and this is using 36 x 3TB SATA drives
    in RAID6 under my GlusterFS volumes, the volumes are replica 2 in my
    case. I haven’t done rebuild tests under load on the RAID6 arrays,
    but I will be doing that later this week. If it takes weeks, as the
    Rackspace documentation you linked to indicates, then yeah that will
    be terrible. I have slightly smaller RAID6 arrays that can rebuild
    in a matter of hours, so I’m hopeful. Hope this helps…

17. Patrick McShane says:

    [March 4, 2013 at 1:33
    pm](http://www.gluster.org/2012/09/howto-using-ufo-swift-a-quick-and-dirty-setup-guide/comment-page-1/#comment-34146)  [(Edit)](http://www.gluster.org/wp-admin/comment.php?action=editcomment&c=34146 "Edit comment")

    Hi there,

    The gist of the problem we’re seeing occurs when we’re doing test
    uploads of file parts using curl (i.e. foo/test\_file.dat/01,
    foo/test\_file.dat/02, …) and using the “X-Object-Manifest” header
    to allow for downloading the parts as a single file.
    Uploads/downloads (from 1GB to 20GB in size) of individual files
    always works just fine.

    It’s only when we try to bind the parts using X-Object-Manifest that
    we’re unable to download them back into to a single large file.

    Curl reports a 503 error immediately when the multipart download is
    attempted.

    We’re using openstack folsom build running on CentOS 6.3 x86\_64
    (using KVM – from the EPEL repo).

    See curl commands and resulting tracebacks in /var/log/messages
    below.

    Here is our package recipe on CentOS 6.3 x86\_64.\
     [root@sd-pigshead \~]\# rpm -qa|grep glust\
     glusterfs-fuse-3.3.1-10.el6.x86\_64\
     glusterfs-swift-proxy-3.3.1-10.el6.noarch\
     glusterfs-3.3.1-10.el6.x86\_64\
     glusterfs-swift-container-3.3.1-10.el6.noarch\
     glusterfs-ufo-3.3.1-10.el6.noarch\
     glusterfs-swift-3.3.1-10.el6.noarch\
     glusterfs-swift-account-3.3.1-10.el6.noarch\
     glusterfs-swift-object-3.3.1-10.el6.noarch\
     glusterfs-server-3.3.1-10.el6.x86\_64\
     [root@sd-pigshead \~]\# rpm -qa|egrep “openstack|glust”\
     glusterfs-fuse-3.3.1-10.el6.x86\_64\
     openstack-glance-2012.2.3-1.el6.noarch\
     openstack-nova-objectstore-2012.2.2-1.el6.noarch\
     openstack-utils-2013.1-1.el6.noarch\
     glusterfs-swift-proxy-3.3.1-10.el6.noarch\
     openstack-nova-volume-2012.2.2-1.el6.noarch\
     openstack-nova-2012.2.2-1.el6.noarch\
     openstack-keystone-2012.2.1-1.el6.noarch\
     glusterfs-3.3.1-10.el6.x86\_64\
     glusterfs-swift-container-3.3.1-10.el6.noarch\
     glusterfs-ufo-3.3.1-10.el6.noarch\
     openstack-nova-common-2012.2.2-1.el6.noarch\
     openstack-nova-api-2012.2.2-1.el6.noarch\
     python-django-openstack-auth-1.0.2-3.el6.noarch\
     openstack-dashboard-2012.2.1-1.el6.noarch\
     glusterfs-swift-3.3.1-10.el6.noarch\
     glusterfs-swift-account-3.3.1-10.el6.noarch\
     openstack-nova-console-2012.2.2-1.el6.noarch\
     openstack-nova-cert-2012.2.2-1.el6.noarch\
     openstack-quantum-2012.2.1-1.el6.noarch\
     glusterfs-swift-object-3.3.1-10.el6.noarch\
     openstack-nova-scheduler-2012.2.2-1.el6.noarch\
     openstack-cinder-2012.2.1-1.el6.noarch\
     glusterfs-server-3.3.1-10.el6.x86\_64\
     openstack-nova-network-2012.2.2-1.el6.noarch\
     openstack-nova-compute-2012.2.2-1.el6.noarch

    curl -v -X PUT -H “X-Auth-Token: \${AUTHTOKEN}” -H “Content-Length:
    \${FILELEN}” \$URLBASE/\${DIRNAME}/\`basename \${FILENAME}\`/00
    –data-binary @\${FILENAME}\_00\
     curl -v -X PUT -H “X-Auth-Token: \${AUTHTOKEN}” -H “Content-Length:
    \${FILELEN}” \$URLBASE/\${DIRNAME}/\`basename \${FILENAME}\`/01
    –data-binary @\${FILENAME}\_01\
     …

    curl -v -X PUT -H “X-Auth-Token: \${AUTHTOKEN}” -H ‘Content-Length:
    0′ -H “X-Object-Manifest: \$DIRNAME/\`basename \$FILENAME\`/”
    \$URLBASE/\$DIRNAME/\`basename \$FILENAME\` –data-binary ”

    curl -v -H “X-Auth-Token: \${AUTHTOKEN}”
    \$URLBASE/\${DIRNAME}/\`basename \${FILENAME}\` \>\`basename
    \$FILENAME\`\_downloaded

    \#\#\#\#\#\#\#\#\# /var/log/messages \#\#\#\#\#\#\#\#\#\#

    Mar 4 10:24:01 sd-pigshead container-server 127.0.0.1 – -
    [04/Mar/2013:10:24:01 +0000] “PUT
    /data-volume/0/AUTH\_data-volume/create\_dir\_test/test\_file.dat”
    201 – “tx9766153aca824b77954cd96b996645c2″ “-” “-” 0.0010\
     Mar 4 10:24:01 sd-pigshead object-server 127.0.0.1 – -
    [04/Mar/2013:10:24:01 +0000] “PUT
    /data-volume/0/AUTH\_data-volume/create\_dir\_test/test\_file.dat”
    201 – “-” “tx9766153aca824b77954cd96b996645c2″ “curl/7.19.7
    (x86\_64-redhat-linux-gnu) libcurl/7.19.7 NSS/3.13.1.0 zlib/1.2.3
    libidn/1.18 libssh2/1.2.2″ 0.0450\
     Mar 4 10:24:01 sd-pigshead container-server 127.0.0.1 – -
    [04/Mar/2013:10:24:01 +0000] “GET
    /data-volume/0/AUTH\_data-volume/create\_dir\_test” 200 –
    “txa396989286234612969f82de55ef9862″ “-” “curl/7.19.7
    (x86\_64-redhat-linux-gnu) libcurl/7.19.7 NSS/3.13.1.0 zlib/1.2.3
    libidn/1.18 libssh2/1.2.2″ 0.0224\
     Mar 4 10:24:01 sd-pigshead object-server 127.0.0.1 – -
    [04/Mar/2013:10:24:01 +0000] “GET
    /data-volume/0/AUTH\_data-volume/create\_dir\_test/test\_file.dat”
    200 – “-” “tx029ac6fcfb464d199bd92e1b1393a508″ “curl/7.19.7
    (x86\_64-redhat-linux-gnu) libcurl/7.19.7 NSS/3.13.1.0 zlib/1.2.3
    libidn/1.18 libssh2/1.2.2″ 0.0009\
     Mar 4 10:24:01 sd-pigshead proxy-server ERROR with Object server
    127.0.0.1:6010/data-volume re: Trying to GET
    /v1/AUTH\_data-volume/create\_dir\_test/test\_file.dat:
    \#012Traceback (most recent call last):\#012 File
    “/usr/lib/python2.6/site-packages/swift/proxy/controllers/base.py”,
    line 595, in GETorHEAD\_base\#012 possible\_source =
    conn.getresponse()\#012 File
    “/usr/lib/python2.6/site-packages/swift/common/bufferedhttp.py”,
    line 102, in getresponse\#012 response =
    HTTPConnection.getresponse(self)\#012 File
    “/usr/lib64/python2.6/httplib.py”, line 990, in getresponse\#012
    response.begin()\#012 File “/usr/lib64/python2.6/httplib.py”, line
    391, in begin\#012 version, status, reason =
    self.\_read\_status()\#012 File “/usr/lib64/python2.6/httplib.py”,
    line 355, in \_read\_status\#012 raise
    BadStatusLine(line)\#012BadStatusLine (txn:
    tx029ac6fcfb464d199bd92e1b1393a508) (client\_ip: 10.12.33.224)\
     Mar 4 10:24:01 sd-pigshead proxy-server Object GET returning 503
    for [] (txn: tx029ac6fcfb464d199bd92e1b1393a508) (client\_ip:
    10.12.33.224)

18. [johnmark](http://www.gluster.org/) says:

    [March 4, 2013 at 7:39
    pm](http://www.gluster.org/2012/09/howto-using-ufo-swift-a-quick-and-dirty-setup-guide/comment-page-1/#comment-34227)  [(Edit)](http://www.gluster.org/wp-admin/comment.php?action=editcomment&c=34227 "Edit comment")

    Patrick – I would recommend you bring this up in \#gluster on IRC or
    on the gluster-users mailing list.

19. Andreas Calvo says:

    [April 11, 2013 at 6:59
    am](http://www.gluster.org/2012/09/howto-using-ufo-swift-a-quick-and-dirty-setup-guide/comment-page-1/#comment-40514)  [(Edit)](http://www.gluster.org/wp-admin/comment.php?action=editcomment&c=40514 "Edit comment")

    When uploading large files, it seems that swift is failing:\
     Apr 11 15:56:16 DFS1 proxy-server ERROR with Object server
    127.0.0.1:6010/testvol re: Trying to write to
    /v1/AUTH\_testvol/vol2/HP\_Service\_Pack\_for\_Proliant\_2012.10.0-0\_713293-001\_spp\_2012.10.0-SPP2012100.2012\_1005.37.iso:
    ChunkWriteTimeout (10s)\
     Apr 11 15:56:16 DFS1 proxy-server Object PUT exceptions during
    send, 0/1 required connections (txn:
    tx3552094ea43b467ab88a53075e32cb3a) (client\_ip: 10.0.96.43)

    Using 2 nodes (2 bricks replica) with 1 proxy server.

    Is it necessary to tweak swift to allow uploading large files?

20. kkeithley says:

    [April 12, 2013 at 7:49
    am](http://www.gluster.org/2012/09/howto-using-ufo-swift-a-quick-and-dirty-setup-guide/comment-page-1/#comment-40642)  [(Edit)](http://www.gluster.org/wp-admin/comment.php?action=editcomment&c=40642 "Edit comment")

    As indicated above (14 Jan 2013), Gluster-Swift comes out-of-the-box
    with the file size limit disabled.

    If you’re using 3.4.0alpha or 3.4.0alpha2, Gluster-Swift is broken
    in those releases.

### Leave a Reply [Cancel reply](/2012/09/howto-using-ufo-swift-a-quick-and-dirty-setup-guide/#respond) {#reply-title}

Logged in as [johnmark](http://www.gluster.org/wp-admin/profile.php).
[Log
out?](http://www.gluster.org/wp-login.php?action=logout&redirect_to=http%3A%2F%2Fwww.gluster.org%2F2012%2F09%2Fhowto-using-ufo-swift-a-quick-and-dirty-setup-guide%2F&_wpnonce=849df536fd "Log out of this account")

Comment

You may use these HTML tags and attributes:
`<a href="" title=""> <abbr title=""> <acronym title=""> <b> <blockquote cite=""> <cite> <code> <del datetime=""> <em> <i> <q cite=""> <strike> <strong> `

###### Recent Posts

-   [Free
    Cache](http://tropicaldevel.wordpress.com/2013/04/09/free-cache/ "Free Cache")
-   [Knowing when to release and deploy your code (…and a mini
    script)](https://ttboj.wordpress.com/2013/04/08/knowing-when-to-release-and-deploy-your-code-and-a-mini-script/ "Knowing when to release and deploy your code (…and a mini script)")
-   [Gluster Testing
    Framework](http://www.gluster.org/2013/04/gluster-testing-framework/ "Gluster Testing Framework")
-   [Turning micro-commits into one
    megacommit](http://www.gluster.org/2013/04/turning-micro-commits-into-one-megacommit-2/ "Turning micro-commits into one megacommit")
-   [Turning micro-commits into one
    megacommit](http://www.gluster.org/2013/04/turning-micro-commits-into-one-megacommit/ "Turning micro-commits into one megacommit")

###### Syndicated Blogs

This is a planet site. See all the blogs syndicated here:

-   [Celingest Blog – Feel the Cloud »
    GlusterFS](http://blog.celingest.com/en/ "Amazon Web Services – APN Advanced Consulting Partner")
-   [conrey.org »
    gluster](http://conrey.org "fermentation and virtualization madness")
-   [Developer in Charge » gluster](http://devincharge.com)
-   [Est modus matulae](http://haerench.blogspot.com/)
-   [Gluster
    Hacker](http://glusterhacker.blogspot.com/ "My GlusterFS Anecdotes")
-   [Gluster | Learner \> log](http://raghavendratalur.in/glusterblog)
-   [HekaFS](http://hekafs.org "Formerly known as CloudFS")
-   [HowtoForge – Linux Howtos and Tutorials
    –](http://www.howtoforge.com "HowtoForge provides user-friendly Linux tutorials about almost every topic.

    If you’ve written a Linux tutorial that you’d like to share, you can contribute it. If you’d like to discuss Linux-related problems, you can use our forum. If you have questions,")
-   [jayunit100](http://jayunit100.blogspot.com/ "statelessly yours")
-   [jebpages »
    gluster](http://blog.jebpages.com "the official blog of blog.jebpages.com")
-   [Joe Julian's
    Blog](http://joejulian.name/blog/feeds/rss/ "The public stuff I’m working on…")
-   [Middleswarth – gluster](http://www.middleswarth.net/topic/gluster)
-   [Nixpanic’s Blog](http://blog.nixpanic.net/search/label/Gluster)
-   [TechnoSophos –
    gluster](http://technosophos.com/taxonomy/term/187/0)
-   [The Technical Blog of James »
    gluster](https://ttboj.wordpress.com "Technical discussion, articles and writeups!")
-   [Tropical Devel »
    OpenStack](http://tropicaldevel.wordpress.com "Bits of code from the pacific")
-   [Vijay
    Bellur](http://vbellur.wordpress.com "Life, Technology and a little more…")

###### Categories

-   [12.10](http://www.gluster.org/category/12-10/ "View all posts filed under 12.10")
-   [3.4](http://www.gluster.org/category/3-4/ "View all posts filed under 3.4")
-   [Android](http://www.gluster.org/category/android/ "View all posts filed under Android")
-   [AWS
    @en](http://www.gluster.org/category/aws-en/ "View all posts filed under AWS @en")
-   [bash](http://www.gluster.org/category/bash/ "View all posts filed under bash")
-   [CentOS](http://www.gluster.org/category/centos/ "View all posts filed under CentOS")
-   [cloud](http://www.gluster.org/category/cloud/ "View all posts filed under cloud")
-   [Debian](http://www.gluster.org/category/debian/ "View all posts filed under Debian")
-   [Development](http://www.gluster.org/category/development/ "View all posts filed under Development")
-   [devops](http://www.gluster.org/category/devops/ "View all posts filed under devops")
-   [dlm](http://www.gluster.org/category/dlm/ "View all posts filed under dlm")
-   [EBS
    @en](http://www.gluster.org/category/ebs-en/ "View all posts filed under EBS @en")
-   [EC2
    @en](http://www.gluster.org/category/ec2-en/ "View all posts filed under EC2 @en")
-   [Events](http://www.gluster.org/category/events/ "View all posts filed under Events")
-   [fedora](http://www.gluster.org/category/fedora/ "View all posts filed under fedora")
-   [file
    descriptor](http://www.gluster.org/category/file-descriptor/ "View all posts filed under file descriptor")
-   [FIXME](http://www.gluster.org/category/fixme/ "View all posts filed under FIXME")
-   [FOSS](http://www.gluster.org/category/foss/ "View all posts filed under FOSS")
-   [Front](http://www.gluster.org/category/front/ "View all posts filed under Front")
-   [git](http://www.gluster.org/category/git/ "View all posts filed under git")
-   [github](http://www.gluster.org/category/github/ "View all posts filed under github")
-   [gluster](http://www.gluster.org/category/gluster/ "View all posts filed under gluster")
-   [Gluster
    Info](http://www.gluster.org/category/gluster-info/ "View all posts filed under Gluster Info")
-   [glusterfs](http://www.gluster.org/category/glusterfs/ "View all posts filed under glusterfs")
-   [google compute
    engine](http://www.gluster.org/category/google-compute-engine/ "View all posts filed under google compute engine")
-   [hack](http://www.gluster.org/category/hack/ "View all posts filed under hack")
-   [High-Availability](http://www.gluster.org/category/high-availability/ "View all posts filed under High-Availability")
-   [Howtos](http://www.gluster.org/category/howtos/ "View all posts filed under Howtos")
-   [hpcloud](http://www.gluster.org/category/hpcloud/ "View all posts filed under hpcloud")
-   [i-use-tags-instead-of-categories-sorry](http://www.gluster.org/category/i-use-tags-instead-of-categories-sorry/ "View all posts filed under i-use-tags-instead-of-categories-sorry")
-   [infrastructure](http://www.gluster.org/category/infrastructure/ "View all posts filed under infrastructure")
-   [iOS](http://www.gluster.org/category/ios/ "View all posts filed under iOS")
-   [IPv6](http://www.gluster.org/category/ipv6/ "View all posts filed under IPv6")
-   [Joomla!
    Plugins](http://www.gluster.org/category/joomla-plugins/ "View all posts filed under Joomla! Plugins")
-   [keepalived](http://www.gluster.org/category/keepalived/ "View all posts filed under keepalived")
-   [KVM](http://www.gluster.org/category/kvm/ "View all posts filed under KVM")
-   [Linux](http://www.gluster.org/category/linux/ "View all posts filed under Linux")
-   [Linux
    @en](http://www.gluster.org/category/linux-en/ "View all posts filed under Linux @en")
-   [make](http://www.gluster.org/category/make/ "View all posts filed under make")
-   [mount](http://www.gluster.org/category/mount/ "View all posts filed under mount")
-   [News](http://www.gluster.org/category/news/ "View all posts filed under News")
-   [Nexenta](http://www.gluster.org/category/nexenta/ "View all posts filed under Nexenta")
-   [open
    source](http://www.gluster.org/category/open-source/ "View all posts filed under open source")
-   [openstack](http://www.gluster.org/category/openstack/ "View all posts filed under openstack")
-   [OpenVZ](http://www.gluster.org/category/openvz/ "View all posts filed under OpenVZ")
-   [ovirt](http://www.gluster.org/category/ovirt/ "View all posts filed under ovirt")
-   [Performance](http://www.gluster.org/category/performance/ "View all posts filed under Performance")
-   [pgo](http://www.gluster.org/category/pgo/ "View all posts filed under pgo")
-   [post](http://www.gluster.org/category/post/ "View all posts filed under post")
-   [Puppet](http://www.gluster.org/category/puppet/ "View all posts filed under Puppet")
-   [Python](http://www.gluster.org/category/python/ "View all posts filed under Python")
-   [rackspace](http://www.gluster.org/category/rackspace/ "View all posts filed under rackspace")
-   [Rants &
    Raves](http://www.gluster.org/category/rants-raves/ "View all posts filed under Rants & Raves")
-   [review](http://www.gluster.org/category/review/ "View all posts filed under review")
-   [RHEL](http://www.gluster.org/category/rhel/ "View all posts filed under RHEL")
-   [Samba](http://www.gluster.org/category/samba/ "View all posts filed under Samba")
-   [Software
    Development](http://www.gluster.org/category/software-development/ "View all posts filed under Software Development")
-   [Storage](http://www.gluster.org/category/storage/ "View all posts filed under Storage")
-   [swift](http://www.gluster.org/category/swift/ "View all posts filed under swift")
-   [Syndicated](http://www.gluster.org/category/syndicated/ "View all posts filed under Syndicated")
-   [system
    administration](http://www.gluster.org/category/system-administration/ "View all posts filed under system administration")
-   [tail
    -f](http://www.gluster.org/category/tail-f/ "View all posts filed under tail -f")
-   [Tech](http://www.gluster.org/category/tech/ "View all posts filed under Tech")
-   [TODO](http://www.gluster.org/category/todo/ "View all posts filed under TODO")
-   [Tutorial](http://www.gluster.org/category/tutorial/ "View all posts filed under Tutorial")
-   [Ubuntu](http://www.gluster.org/category/ubuntu/ "View all posts filed under Ubuntu")
-   [Uncategorized](http://www.gluster.org/category/uncategorized/ "View all posts filed under Uncategorized")
-   [user
    story](http://www.gluster.org/category/user-story/ "View all posts filed under user story")
-   [vip](http://www.gluster.org/category/vip/ "View all posts filed under vip")
-   [VirtualBox](http://www.gluster.org/category/virtualbox/ "View all posts filed under VirtualBox")
-   [Virtualization](http://www.gluster.org/category/virtualization/ "View all posts filed under Virtualization")
-   [volumes](http://www.gluster.org/category/volumes/ "View all posts filed under volumes")
-   [vrrp](http://www.gluster.org/category/vrrp/ "View all posts filed under vrrp")
-   [Web
    Server](http://www.gluster.org/category/web-server/ "View all posts filed under Web Server")
-   [XXX](http://www.gluster.org/category/xxx/ "View all posts filed under XXX")

###### Tags

-   [alpha](http://www.gluster.org/tag/alpha/ "alpha Tag")
-   [awards](http://www.gluster.org/tag/awards/ "awards Tag")
-   [best in
    show](http://www.gluster.org/tag/best-in-show/ "best in show Tag")
-   [cifs](http://www.gluster.org/tag/cifs/ "cifs Tag")
-   [citrix
    synergy](http://www.gluster.org/tag/citrix-synergy/ "citrix synergy Tag")
-   [client
    mount](http://www.gluster.org/tag/client-mount/ "client mount Tag")
-   [debian](http://www.gluster.org/tag/debian-2/ "debian Tag")
-   [Developers](http://www.gluster.org/tag/developers/ "Developers Tag")
-   [featured](http://www.gluster.org/tag/featured/ "featured Tag")
-   [glusterfs](http://www.gluster.org/tag/glusterfs/ "glusterfs Tag")
-   [Jeff
    Darcy](http://www.gluster.org/tag/jeff-darcy/ "Jeff Darcy Tag")
-   [linuxcon](http://www.gluster.org/tag/linuxcon/ "linuxcon Tag")
-   [new
    releases](http://www.gluster.org/tag/new-releases/ "new releases Tag")
-   [nfs](http://www.gluster.org/tag/nfs/ "nfs Tag")
-   [pthree](http://www.gluster.org/tag/pthree/ "pthree Tag")
-   [Python](http://www.gluster.org/tag/python/ "Python Tag")
-   [Q&A](http://www.gluster.org/tag/qa/ "Q&A Tag")
-   [saas cloud computing cloud storage drupal gardens drupal acquia
    gluster](http://www.gluster.org/tag/saas-cloud-computing-cloud-storage-drupal-gardens-drupal-acquia-gluster/ "saas cloud computing cloud storage drupal gardens drupal acquia gluster Tag")
-   [samba](http://www.gluster.org/tag/samba-2/ "samba Tag")
-   [small
    files](http://www.gluster.org/tag/small-files/ "small files Tag")
-   [storage
    volume](http://www.gluster.org/tag/storage-volume/ "storage volume Tag")
-   [synergy
    buzz](http://www.gluster.org/tag/synergy-buzz/ "synergy buzz Tag")
-   [testing](http://www.gluster.org/tag/testing/ "testing Tag")
-   [tom
    trainer](http://www.gluster.org/tag/tom-trainer/ "tom trainer Tag")
-   [trends](http://www.gluster.org/tag/trends/ "trends Tag")
-   [user
    stories](http://www.gluster.org/tag/user-stories/ "user stories Tag")
-   [video](http://www.gluster.org/tag/video/ "video Tag")
-   [winning](http://www.gluster.org/tag/winning/ "winning Tag")
-   [xen](http://www.gluster.org/tag/xen/ "xen Tag")

### Upcoming Events

### User Groups and Meetups

-   [See upcoming events](/meetups/)
-   [Start a GlusterFS community](http://www.meetup.com/glusterfs/)

### GlusterFS 3.3 Now Available

-   Object storage, HDFS compatibility, Proactive self-healing
-   Plus bug fixes and performance enhancements
-   [**Download now**](/download/)

Copyright © 2012 Red Hat, Inc. All Rights Reserved | [Legal &
Privacy](http://www.gluster.org/legal/)

[![Facebook](http://www.gluster.org/wp-content/themes/gluster1/images/facebook.png)](http://www.facebook.com/GlusterInc)
[![Twitter](http://www.gluster.org/wp-content/themes/gluster1/images/twitter.png)](http://twitter.com/GlusterOrg)
[![Youtube](http://www.gluster.org/wp-content/themes/gluster1/images/youtube.png)](http://www.youtube.com/GlusterStorage)
[![LinkedIn](http://www.gluster.org/wp-content/themes/gluster1/images/linkedin.png)](#)
[![Slideshare](http://www.gluster.org/wp-content/themes/gluster1/images/slideshare.png)](http://www.slideshare.net/Gluster)

Thank you
---------

Your feedback have been received.

[![Close](http://www.gluster.org/wp-content/plugins/usernoise/images/ok.png)](#)

[Skip to toolbar](#wp-toolbar)

-   [](http://www.gluster.org/wp-admin/about.php "About WordPress")
    -   [About WordPress](http://www.gluster.org/wp-admin/about.php)

    -   [WordPress.org](http://wordpress.org/)
    -   [Documentation](http://codex.wordpress.org/)
    -   [Support Forums](http://wordpress.org/support/)
    -   [Feedback](http://wordpress.org/support/forum/requests-and-feedback)

-   [Gluster Community Website](http://www.gluster.org/wp-admin/)
    -   [Dashboard](http://www.gluster.org/wp-admin/)

    -   [Themes](http://www.gluster.org/wp-admin/themes.php)
    -   [Customize](http://www.gluster.org/wp-admin/customize.php?url=http%3A%2F%2Fwww.gluster.org%2F2012%2F09%2Fhowto-using-ufo-swift-a-quick-and-dirty-setup-guide%2F)
    -   [Widgets](http://www.gluster.org/wp-admin/widgets.php)
    -   [Menus](http://www.gluster.org/wp-admin/nav-menus.php)

-   [Usernoise
    13](http://www.gluster.org/wp-admin/edit.php?order=desc&post_type=un_feedback&post_status=pending)
    -   [Feedback](http://www.gluster.org/wp-admin/edit.php?order=desc&post_type=un_feedback&post_status=pending)
    -   [Settings](http://www.gluster.org/wp-admin/options-general.php?page=usernoise)

-   [1](http://www.gluster.org/wp-admin/edit-comments.php "1 comment awaiting moderation")
-   [New](http://www.gluster.org/wp-admin/post-new.php "Add New")
    -   [Post](http://www.gluster.org/wp-admin/post-new.php)
    -   [Media](http://www.gluster.org/wp-admin/media-new.php)
    -   [Link](http://www.gluster.org/wp-admin/link-add.php)
    -   [Page](http://www.gluster.org/wp-admin/post-new.php?post_type=page)
    -   [User](http://www.gluster.org/wp-admin/user-new.php)

-   [Edit
    Post](http://www.gluster.org/wp-admin/post.php?post=1412&action=edit)

-   -   [Howdy,
    johnmark](http://www.gluster.org/wp-admin/profile.php "My Account")
    -   [johnmark](http://www.gluster.org/wp-admin/profile.php)
    -   [Edit My Profile](http://www.gluster.org/wp-admin/profile.php)
    -   [Log
        Out](http://www.gluster.org/wp-login.php?action=logout&_wpnonce=849df536fd)

[Log
Out](http://www.gluster.org/wp-login.php?action=logout&_wpnonce=849df536fd)
