###### Configuring basic networking
<kbd>(both)</kbd> `echo $'192.168.99.11 node1\n192.168.99.12 node2' >> /etc/hosts`

###### Installing packages
<kbd>(both)</kbd> `yum install -y pacemaker pcs vim`

###### Setting firewall rules
<kbd>(both)</kbd> `systemctl start firewalld`
<kbd>(both)</kbd> `systemctl enable firewalld`
<kbd>(both)</kbd> `firewall-cmd --permanent --add-service=nfs`
<kbd>(both)</kbd> `firewall-cmd --permanent --add-service=rpc-bind`
<kbd>(both)</kbd> `firewall-cmd --permanent --add-service=mountd`
<kbd>(both)</kbd> `firewall-cmd --permanent --add-service=high-availability`

<kbd>(node1)</kbd> `firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="192.168.99.12" port port="7789" protocol="tcp" accept'`
<kbd>(node1)</kbd> `firewall-cmd --reload`
<kbd>(node1)</kbd> `hostnamectl set-hostname node1`

<kbd>(node2)</kbd> `firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="192.168.99.11" port port="7789" protocol="tcp" accept'`
<kbd>(node2)</kbd> `firewall-cmd --reload`
<kbd>(node2)</kbd> `hostnamectl set-hostname node2`

###### Disabling SELINUX (not bp)
<kbd>(both)</kbd> `vim /etc/sysconfig/selinux "SELINUX=disabled"`

###### Configuring Peacemaker
<kbd>(both)</kbd> `echo "H@xorP@assWD" | passwd hacluster --stdin`
<kbd>(both)</kbd> `systemctl start pcsd`
<kbd>(both)</kbd> `systemctl enable pcsd`

<kbd>(node1)</kbd> `pcs cluster auth node1 node2 -u hacluster -p H@xorP@assWD`
<kbd>(node1)</kbd> `pcs cluster setup --start --name mycluster node1 node2`
<kbd>(node1)</kbd> `systemctl start corosync`
<kbd>(node1)</kbd> `systemctl enable corosync`
<kbd>(node1)</kbd> `pcs cluster start --all`
<kbd>(node1)</kbd> `pcs cluster enable --all`

<kbd>(both)</kbd> `corosync-cfgtool -s`

<kbd>(node1)</kbd> `pcs cluster cib mycluster`

###### Adding properties to PCS cluster
<kbd>(node1)</kbd> `pcs -f /root/mycluster property set no-quorum-policy=ignore`
<kbd>(node1)</kbd> `pcs -f /root/mycluster property set stonith-enabled=false`
<kbd>(node1)</kbd> `pcs -f /root/mycluster resource defaults resource-stickiness=300`

###### Preparing the volume
<kbd>(both)</kbd> `fdisk /dev/sdb`
<kbd>(both)</kbd> `pvcreate /dev/sdb1`
<kbd>(both)</kbd> `vgcreate vg00 /dev/sdb1`
<kbd>(both)</kbd> `lvcreate -l 95%FREE -n drbd-r0 vg00`
<kbd>(both)</kbd> `pvdisplay`
<kbd>(both)</kbd> `ls /dev/mapper/`

###### Installing required packages
<kbd>(both)</kbd> `rpm --import https://www.elrepo.org/RPM-GPG-KEY-elrepo.org`
<kbd>(both)</kbd> `rpm -Uvh http://www.elrepo.org/elrepo-release-7.0-3.el7.elrepo.noarch.rpm`
<kbd>(both)</kbd> `yum install -y kmod-drbd84 drbd84-utils`
<kbd>(both)</kbd> `yum install nfs-utils -y`

<kbd>(both)</kbd> `systemctl stop nfs-lock && systemctl disable nfs-lock`
<kbd>(both)</kbd> `modprobe drbd`
<kbd>(both)</kbd> `echo drbd > /etc/modules-load.d/drbd.conf`

###### Adding DRBD configuration files
<kbd>(both)</kbd> `vim /etc/drbd.conf`
>     include "drbd.d/global_common.conf";
>     include "drbd.d/*.res";
>     global {
>         usage-count no;
>     }
>     resource r0 {
>         protocol C;
>         startup {
>             degr-wfc-timeout 60;
>             outdated-wfc-timeout 30;
>             wfc-timeout 20;
>         }
>         disk {
>             on-io-error detach;
>         }
>         net {
>             cram-hmac-alg sha1;
>             shared-secret "Daveisc00l123313";
>         }
>         on node1 {
>             device /dev/drbd0;
>             disk /dev/mapper/vg00-drbd--r0;
>             address 192.168.99.11:7789;
>             meta-disk internal;
>         }
>         on node2 {
>             device /dev/drbd0;
>             disk /dev/mapper/vg00-drbd--r0;
>             address 192.168.99.12:7789;
>             meta-disk internal;
>         }
>     }

<kbd>(both)</kbd> `vim /etc/drbd.d/global_common.conf`
>     common {
>         handlers {
>         }
>         startup {
>         }
>         options {
>         }
>         disk {
>         }
>         net {
>                 after-sb-0pri discard-zero-changes;
>                 after-sb-1pri discard-secondary; 
>                 after-sb-2pri disconnect;
>         }
>     }

###### Creating DRBD resource
<kbd>(node1)</kbd> `drbdadm create-md r0`
<kbd>(node1)</kbd> `drbdadm up r0`
<kbd>(node1)</kbd> `drbdadm primary r0 --force`
<kbd>(node1)</kbd> `drbdadm -- --overwrite-data-of-peer primary all`
<kbd>(node1)</kbd> `drbdadm outdate r0`
<kbd>(node1)</kbd> `mkfs.ext4 /dev/drbd0`

<kbd>(node2)</kbd> `drbdadm create-md r0`
<kbd>(node2)</kbd> `drbdadm up r0`
<kbd>(node2)</kbd> `drbdadm secondary all`

###### Creating and configuration Peacemaker resources
<kbd>(node1)</kbd> `cd /root`
<kbd>(node1)</kbd> `pcs -f /root/mycluster resource create r0 ocf:linbit:drbd drbd_resource=r0 op monitor interval=10s`
<kbd>(node1)</kbd> `pcs -f /root/mycluster resource master r0-clone r0 master-max=1 master-node-max=1 clone-max=2 clone-node-max=1 notify=true`
<kbd>(node1)</kbd> `pcs -f /root/mycluster resource create drbd-fs Filesystem device="/dev/drbd0" directory="/data" fstype="ext4"`
<kbd>(node1)</kbd> `pcs -f /root/mycluster constraint colocation add drbd-fs with r0-clone INFINITY with-rsc-role=Master`
<kbd>(node1)</kbd> `pcs -f /root/mycluster resource create vip1 ocf:heartbeat:IPaddr2 ip=192.168.99.13 cidr_netmask=24 op monitor interval=10s`
<kbd>(node1)</kbd> `pcs -f /root/mycluster constraint colocation add vip1 with drbd-fs INFINITY`
<kbd>(node1)</kbd> `pcs -f /root/mycluster constraint order drbd-fs then vip1`
<kbd>(node1)</kbd> `pcs -f /root/mycluster resource show`
<kbd>(node1)</kbd> `pcs -f /root/mycluster constraint`
<kbd>(node1)</kbd> `pcs cluster cib-push mycluster`

<kbd>(node1)</kbd> `pcs -f /root/mycluster resource create nfsd nfsserver nfs_shared_infodir=/data/nfsinfo`
<kbd>(node1)</kbd> `pcs -f /root/mycluster resource create nfsroot exportfs clientspec="192.168.99.0/24" options=rw,sync,no_root_squash directory=/data fsid=0`
<kbd>(node1)</kbd> `pcs -f /root/mycluster constraint colocation add nfsd with vip1 INFINITY`
<kbd>(node1)</kbd> `pcs -f /root/mycluster constraint colocation add vip1 with nfsroot INFINITY`
<kbd>(node1)</kbd> `pcs -f /root/mycluster constraint order vip1 then nfsd`
<kbd>(node1)</kbd> `pcs -f /root/mycluster constraint order nfsd then nfsroot`
<kbd>(node1)</kbd> `pcs -f /root/mycluster constraint order promote r0-clone then start drbd-fs`
<kbd>(node1)</kbd> `pcs resource cleanup`
<kbd>(node1)</kbd> `pcs cluster cib-push mycluster`

###### Testing
Switiching to the 2nd
<kbd>(node1)</kbd> `pcs resource move drbd-fs node2`
