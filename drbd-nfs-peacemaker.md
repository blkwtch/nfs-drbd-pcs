(both) echo $'192.168.99.11 node1\n192.168.99.12 node2' >> /etc/hosts
(both) yum install -y pacemaker pcs vim

(both) systemctl start firewalld
(both) systemctl enable firewalld
(both) firewall-cmd --permanent --add-service=nfs
(both) firewall-cmd --permanent --add-service=rpc-bind
(both) firewall-cmd --permanent --add-service=mountd
(both) firewall-cmd --permanent --add-service=high-availability

(node1) firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="192.168.99.12" port port="7789" protocol="tcp" accept'
(node1) firewall-cmd --reload
(node1) hostnamectl set-hostname node1

(node2) firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="192.168.99.11" port port="7789" protocol="tcp" accept'
(node2) firewall-cmd --reload
(node2) hostnamectl set-hostname node2

(both) vim /etc/sysconfig/selinux "SELINUX=disabled"

(both) echo "H@xorP@assWD" | passwd hacluster --stdin
(both) systemctl start pcsd
(both) systemctl enable pcsd

(node1) pcs cluster auth node1 node2 -u hacluster -p H@xorP@assWD
(node1) pcs cluster setup --start --name mycluster node1 node2
(node1) systemctl start corosync
(node1) systemctl enable corosync
(node1) pcs cluster start --all
(node1) pcs cluster enable --all

(both) corosync-cfgtool -s

(node1) pcs cluster cib mycluster

(node1) pcs -f /root/mycluster property set no-quorum-policy=ignore
(node1) pcs -f /root/mycluster property set stonith-enabled=false
(node1) pcs -f /root/mycluster resource defaults resource-stickiness=300

(both) fdisk /dev/sdb
(both) pvcreate /dev/sdb1
(both) vgcreate vg00 /dev/sdb1
(both) lvcreate -l 95%FREE -n drbd-r0 vg00
(both) pvdisplay
(both) ls /dev/mapper/

(both) rpm --import https://www.elrepo.org/RPM-GPG-KEY-elrepo.org
(both) rpm -Uvh http://www.elrepo.org/elrepo-release-7.0-3.el7.elrepo.noarch.rpm
(both) yum install -y kmod-drbd84 drbd84-utils
(both) modprobe drbd
(both) echo drbd > /etc/modules-load.d/drbd.conf
(both) vim /etc/drbd.conf
    include "drbd.d/global_common.conf";
    include "drbd.d/*.res";
    global {
        usage-count no;
    }
    resource r0 {
        protocol C;
        startup {
            degr-wfc-timeout 60;
            outdated-wfc-timeout 30;
            wfc-timeout 20;
        }
        disk {
            on-io-error detach;
        }
        net {
            cram-hmac-alg sha1;
            shared-secret "Daveisc00l123313";
        }
        on node1 {
            device /dev/drbd0;
            disk /dev/mapper/vg00-drbd--r0;
            address 192.168.99.11:7789;
            meta-disk internal;
        }
        on node2 {
            device /dev/drbd0;
            disk /dev/mapper/vg00-drbd--r0;
            address 192.168.99.12:7789;
            meta-disk internal;
        }
    }

(both) vim /etc/drbd.d/global_common.conf
    common {
        handlers {
        }
        startup {
        }
        options {
        }
        disk {
        }
        net {
                after-sb-0pri discard-zero-changes;
                after-sb-1pri discard-secondary; 
                after-sb-2pri disconnect;
        }
    }

(node1) drbdadm create-md r0
(node1) drbdadm up r0
(node1) drbdadm primary r0 --force
(node1) drbdadm -- --overwrite-data-of-peer primary all
(node1) drbdadm outdate r0
(node1) mkfs.ext4 /dev/drbd0

(node2) drbdadm create-md r0
(node2) drbdadm up r0
(node2) drbdadm secondary all

(node1) pcs -f /root/mycluster resource create r0 ocf:linbit:drbd drbd_resource=r0 op monitor interval=10s
(node1) pcs -f /root/mycluster resource master r0-clone r0 master-max=1 master-node-max=1 clone-max=2 clone-node-max=1 notify=true
(node1) pcs -f /root/mycluster resource create drbd-fs Filesystem device="/dev/drbd0" directory="/data" fstype="ext4"
(node1) pcs -f /root/mycluster constraint colocation add drbd-fs with r0-clone INFINITY with-rsc-role=Master
(node1) pcs -f /root/mycluster resource create vip1 ocf:heartbeat:IPaddr2 ip=192.168.99.13 cidr_netmask=24 op monitor interval=10s
(node1) pcs -f /root/mycluster constraint colocation add vip1 with drbd-fs INFINITY
(node1) pcs -f /root/mycluster constraint order drbd-fs then vip1
(node1) pcs -f /root/mycluster resource show
(node1) pcs -f /root/mycluster constraint
(node1) pcs cluster cib-push mycluster

(both) yum install nfs-utils -y
(both) systemctl stop nfs-lock && systemctl disable nfs-lock
(node1) pcs -f /root/mycluster resource create nfsd nfsserver nfs_shared_infodir=/data/nfsinfo
(node1) pcs -f /root/mycluster resource create nfsroot exportfs clientspec="192.168.99.0/24" options=rw,sync,no_root_squash directory=/data fsid=0
(node1) pcs -f /root/mycluster constraint colocation add nfsd with vip1 INFINITY
(node1) pcs -f /root/mycluster constraint colocation add vip1 with nfsroot INFINITY
(node1) pcs -f /root/mycluster constraint order vip1 then nfsd
(node1) pcs -f /root/mycluster constraint order nfsd then nfsroot
(node1) pcs -f /root/mycluster constraint order promote r0-clone then start drbd-fs
(node1) pcs resource cleanup
(node1) pcs cluster cib-push mycluster

(node1) pcs resource move drbd-fs node1