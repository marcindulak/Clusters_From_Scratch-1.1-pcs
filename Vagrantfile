# coding: utf-8
# -*- mode: ruby -*-
# vi: set ft=ruby :
Vagrant.configure(2) do |config|
  config.vm.define "pcmk-2" do |pcmk2|
    pcmk2.vm.box = "chef/fedora-21"
    pcmk2.vm.box_url = "chef/fedora-21"
    # fedora does not use eth1 anymore ... must configure network manually
    # https://github.com/jedi4ever/veewee/issues/970
    pcmk2.vm.network "private_network", ip: "192.168.122.102", auto_config: false
    pcmk2.vm.network "forwarded_port", guest: 22, host: 22102
  end
  config.vm.define "pcmk-1" do |pcmk1|
    pcmk1.vm.box = "chef/fedora-21"
    pcmk1.vm.box_url = "chef/fedora-21"
    # fedora does not use eth1 anymore ... must configure network manually
    # https://github.com/jedi4ever/veewee/issues/970
    pcmk1.vm.network "private_network", ip: "192.168.122.101", auto_config: false
    pcmk1.vm.network "forwarded_port", guest: 22, host: 22101
  end
  # disable IPv6 on Linux
  $linux_disable_ipv6 = <<SCRIPT
sysctl -w net.ipv6.conf.default.disable_ipv6=1
sysctl -w net.ipv6.conf.all.disable_ipv6=1
sysctl -w net.ipv6.conf.lo.disable_ipv6=1
SCRIPT
  # hostnames
  $etc_hosts = <<SCRIPT
cat <<END >> /etc/hosts
192.168.122.101 pcmk-1
192.168.122.102 pcmk-2
END
SCRIPT
  # setenforce 0
  $setenforce_0 = <<SCRIPT
if test `getenforce` = 'Enforcing'; then setenforce 0; fi
#sed -Ei 's/^SELINUX=.*/SELINUX=Permissive/' /etc/selinux/config
SCRIPT
  # configure the second vagrant eth interface
  $ifcfg_device = <<SCRIPT
IPADDR=$1
NETMASK=$2
DEVICE=$3
cat <<END >> /etc/sysconfig/network-scripts/ifcfg-$DEVICE
  NM_CONTROLLED=no
  BOOTPROTO=none
  ONBOOT=yes
  IPADDR=$IPADDR
  NETMASK=$NETMASK
  DEVICE=$DEVICE
  PEERDNS=no
END
ARPCHECK=no /sbin/ifup $DEVICE 2> /dev/null
SCRIPT
  # .ssh permission
  $chmod_600_dotssh = <<SCRIPT
sudo su -c "chmod -R 600 ~$1/.ssh/"
sudo su -c "chmod 700 ~$1/.ssh"
SCRIPT
  # .ssh ownership
  $chown_dotssh = <<SCRIPT
sudo su -c "chown -R $1:$2 ~$1/.ssh/"
SCRIPT
  $apache_index_html = <<SCRIPT
cat <<'END' > /var/www/html/index.html
 <html>
 <body>My Test Site - $(hostname)</body>
 </html>
END
SCRIPT
  $apache_status_conf = <<SCRIPT
cat <<'END' > /etc/httpd/conf.d/status.conf
 <Location /server-status>
    SetHandler server-status
    Order deny,allow
    Deny from all
    Allow from 127.0.0.1
 </Location>
END
SCRIPT
  # create block device /dev/fedora-server_pcmk
  $dev_fedora_server_pcmk = <<SCRIPT
# create a ~2GiB file
dd if=/dev/zero of=/fedora-server_pcmk bs=1024k count=2048
i=777
mknod /dev/fedora-server_pcmk b 7 $i
chown --reference=/dev/loop0 /dev/fedora-server_pcmk
chmod --reference=/dev/loop0 /dev/fedora-server_pcmk
losetup /dev/fedora-server_pcmk /fedora-server_pcmk
SCRIPT
  $etc_drbd_d_wwwdata_res = <<SCRIPT
cat <<'END' > /etc/drbd.d/wwwdata.res
resource wwwdata {
 protocol C;
 # pcmk-2 is not available when pcmk-1 is being provisioned, wait for 5 sec ...
 # startup { degr-wfc-timeout 5; wfc-timeout 5; }
 meta-disk internal;
 device /dev/drbd1;
 syncer {
  verify-alg sha1;
 }
 net {
  allow-two-primaries;
 }
 on pcmk-1 {
  disk   /dev/fedora-server_pcmk-1/drbd-demo;
  address  192.168.122.101:7789;
 }
 on pcmk-2 {
  disk   /dev/fedora-server_pcmk-2/drbd-demo;
  address  192.168.122.102:7789;
 }
}
END
SCRIPT
  $mnt_index_html = <<SCRIPT
cat <<'END' > /mnt/index.html
 <html>
  <body>My Test Site - DRBD</body>
 </html>
END
SCRIPT
  config.vm.define "pcmk-2" do |pcmk2|
    pcmk2.vm.provision :shell, :inline => $linux_disable_ipv6, run: "always"
    pcmk2.vm.provision :shell, :inline => "hostname pcmk-2", run: "always"
    pcmk2.vm.provision :shell, :inline => $setenforce_0, run: "always"
    pcmk2.vm.provision "shell" do |s|
      s.inline = $ifcfg_device
      s.args   = "192.168.122.102 255.255.255.0 enp0s8"
    end
    pcmk2.vm.provision :shell, :inline => "ifup enp0s8", run: "always"
    pcmk2.vm.provision :shell, :inline => $etc_hosts
    pcmk2.vm.provision :file, source: "~/.vagrant.d/insecure_private_key", destination: "~vagrant/.ssh/id_rsa"
    pcmk2.vm.provision :shell, :inline => "cp -rp ~vagrant/.ssh ~root/"
    pcmk2.vm.provision :shell, :inline => "yum install -y wget"
    pcmk2.vm.provision :shell, :inline => "wget --no-check-certificate https://raw.github.com/mitchellh/vagrant/master/keys/vagrant.pub -O ~root/.ssh/authorized_keys"
    pcmk2.vm.provision "shell" do |s|
      s.inline = $chmod_600_dotssh
      s.args   = "root"
    end
    pcmk2.vm.provision "shell" do |s|
      s.inline = $chown_dotssh
      s.args   = "root root"
    end
    pcmk2.vm.provision :shell, :inline => "yum install -y pacemaker pcs"
    pcmk2.vm.provision :shell, :inline => "echo password | passwd --stdin hacluster"
    pcmk2.vm.provision :shell, :inline => "systemctl start pcsd.service"
    pcmk2.vm.provision :shell, :inline => "systemctl enable pcsd.service"
    pcmk2.vm.provision :shell, :inline => "pcs cluster auth -u hacluster -p password pcmk-2"
    pcmk2.vm.provision :shell, :inline => "yum install -y httpd wget"
    pcmk2.vm.provision :shell, :inline => $apache_index_html
    pcmk2.vm.provision :shell, :inline => $apache_status_conf
    pcmk2.vm.provision :shell, :inline => "yum install -y drbd-pacemaker drbd-udev"
    pcmk2.vm.provision :shell, :inline => $dev_fedora_server_pcmk
    pcmk2.vm.provision :shell, :inline => "vgcreate fedora-server_pcmk-2 /dev/fedora-server_pcmk"
    pcmk2.vm.provision :shell, :inline => $etc_drbd_d_wwwdata_res
    pcmk2.vm.provision :shell, :inline => "echo drbd > /etc/modules-load.d/drbd.conf"
    pcmk2.vm.provision :shell, :inline => "yum install -y psmisc"  # FIXME2
    # kernel-modules-extra for the running kernel
    pcmk2.vm.provision :shell, :inline => "yum install -y kernel-modules-extra-`uname -r` kernel-headers-`uname -r`"
    pcmk2.vm.provision :shell, :inline => "yum install -y gfs2-utils dlm"
  end
  config.vm.define "pcmk-1" do |pcmk1|
    pcmk1.vm.provision :shell, :inline => $linux_disable_ipv6, run: "always"
    pcmk1.vm.provision :shell, :inline => "hostname pcmk-1", run: "always"
    pcmk1.vm.provision :shell, :inline => $setenforce_0, run: "always"
    pcmk1.vm.provision "shell" do |s|
      s.inline = $ifcfg_device
      s.args   = "192.168.122.101 255.255.255.0 enp0s8"
    end
    pcmk1.vm.provision :shell, :inline => "ifup enp0s8", run: "always"
    pcmk1.vm.provision :shell, :inline => $etc_hosts
    pcmk1.vm.provision :file, source: "~/.vagrant.d/insecure_private_key", destination: "~vagrant/.ssh/id_rsa"
    pcmk1.vm.provision :shell, :inline => "cp -rp ~vagrant/.ssh ~root/"
    pcmk1.vm.provision :shell, :inline => "yum install -y wget"
    pcmk1.vm.provision :shell, :inline => "wget --no-check-certificate https://raw.github.com/mitchellh/vagrant/master/keys/vagrant.pub -O ~root/.ssh/authorized_keys"
    pcmk1.vm.provision "shell" do |s|
      s.inline = $chmod_600_dotssh
      s.args   = "root"
    end
    pcmk1.vm.provision "shell" do |s|
      s.inline = $chown_dotssh
      s.args   = "root root"
    end
    pcmk1.vm.provision :shell, :inline => "yum install -y pacemaker pcs"
    pcmk1.vm.provision :shell, :inline => "echo password | passwd --stdin hacluster"
    pcmk1.vm.provision :shell, :inline => "systemctl start pcsd.service"
    pcmk1.vm.provision :shell, :inline => "systemctl enable pcsd.service"
    pcmk1.vm.provision :shell, :inline => "pcs cluster auth -u hacluster -p password pcmk-1 pcmk-2"
    pcmk1.vm.provision :shell, :inline => "yum install -y psmisc"  # FIXME1
    pcmk1.vm.provision :shell, :inline => "pcs cluster setup --name mycluster pcmk-1 pcmk-2"
    pcmk1.vm.provision :shell, :inline => "pcs cluster start --all"
    pcmk1.vm.provision :shell, :inline => "corosync-cfgtool -s"
    pcmk1.vm.provision :shell, :inline => "corosync-cmapctl  | grep members"
    pcmk1.vm.provision :shell, :inline => "pcs status corosync"
    pcmk1.vm.provision :shell, :inline => "pstree -p `pidof pacemakerd`"
    pcmk1.vm.provision :shell, :inline => "pcs status"
    pcmk1.vm.provision :shell, :inline => "journalctl | grep -i error"
    pcmk1.vm.provision :shell, :inline => "pcs cluster cib"
    pcmk1.vm.provision :shell, :inline => "! crm_verify -L -V"
    pcmk1.vm.provision :shell, :inline => "pcs property set stonith-enabled=false"
    pcmk1.vm.provision :shell, :inline => "crm_verify -L"
    pcmk1.vm.provision :shell, :inline => "pcs resource create ClusterIP ocf:heartbeat:IPaddr2 ip=192.168.122.120 cidr_netmask=32 op monitor interval=30s"
    pcmk1.vm.provision :shell, :inline => "pcs resource standards"
    pcmk1.vm.provision :shell, :inline => "pcs resource providers"
    pcmk1.vm.provision :shell, :inline => "pcs resource agents ocf:heartbeat"
    pcmk1.vm.provision :shell, :inline => "pcs cluster stop `pcs status | grep ClusterIP | cut -d' ' -f3`"
    pcmk1.vm.provision :shell, :inline => "! pcs status"
    pcmk1.vm.provision :shell, :inline => "ssh -o StrictHostKeyChecking=no pcmk-2 'pcs status'"
    pcmk1.vm.provision :shell, :inline => "pcs cluster start pcmk-1"
    pcmk1.vm.provision :shell, :inline => "pcs resource defaults resource-stickiness=100"
    pcmk1.vm.provision :shell, :inline => "pcs resource defaults"
    pcmk1.vm.provision :shell, :inline => "yum install -y httpd wget"
    pcmk1.vm.provision :shell, :inline => $apache_index_html
    pcmk1.vm.provision :shell, :inline => $apache_status_conf
    pcmk1.vm.provision :shell, :inline => "pcs resource create WebSite ocf:heartbeat:apache configfile=/etc/httpd/conf/httpd.conf statusurl='http://localhost/server-status' op monitor interval=1min"
    pcmk1.vm.provision :shell, :inline => "pcs resource op defaults timeout=240s"
    pcmk1.vm.provision :shell, :inline => "pcs resource op defaults"
    pcmk1.vm.provision :shell, :inline => "pcs status"
    pcmk1.vm.provision :shell, :inline => "wget -O - http://127.0.0.1/server-status"
    pcmk1.vm.provision :shell, :inline => "pcs constraint colocation add WebSite with ClusterIP INFINITY"
    pcmk1.vm.provision :shell, :inline => "pcs constraint"
    pcmk1.vm.provision :shell, :inline => "pcs status"
    pcmk1.vm.provision :shell, :inline => "pcs constraint order ClusterIP then WebSite"
    pcmk1.vm.provision :shell, :inline => "pcs constraint"
    pcmk1.vm.provision :shell, :inline => "pcs constraint location WebSite prefers pcmk-1=50"
    pcmk1.vm.provision :shell, :inline => "pcs constraint"
    pcmk1.vm.provision :shell, :inline => "pcs status"
    pcmk1.vm.provision :shell, :inline => "crm_simulate -sL"
    pcmk1.vm.provision :shell, :inline => "pcs constraint location WebSite prefers pcmk-1=INFINITY"
    pcmk1.vm.provision :shell, :inline => "pcs constraint"
    pcmk1.vm.provision :shell, :inline => "pcs status"
    pcmk1.vm.provision :shell, :inline => "pcs constraint --full"
    pcmk1.vm.provision :shell, :inline => "pcs constraint remove location-WebSite-pcmk-1-INFINITY"
    pcmk1.vm.provision :shell, :inline => "pcs constraint"
    pcmk1.vm.provision :shell, :inline => "pcs status"
    pcmk1.vm.provision :shell, :inline => "yum install -y drbd-pacemaker drbd-udev"
    pcmk1.vm.provision :shell, :inline => $dev_fedora_server_pcmk
    pcmk1.vm.provision :shell, :inline => "vgcreate fedora-server_pcmk-1 /dev/fedora-server_pcmk"
    pcmk1.vm.provision :shell, :inline => $etc_drbd_d_wwwdata_res
    pcmk1.vm.provision :shell, :inline => "vgdisplay | grep -e Name -e Free"
    pcmk1.vm.provision :shell, :inline => "lvcreate --name drbd-demo --size 1G fedora-server_pcmk-1"
    pcmk1.vm.provision :shell, :inline => "ssh -o StrictHostKeyChecking=no pcmk-2 'lvcreate --name drbd-demo --size 1G fedora-server_pcmk-2'"
    pcmk1.vm.provision :shell, :inline => "ssh -o StrictHostKeyChecking=no pcmk-2 'drbdadm create-md wwwdata'"
    pcmk1.vm.provision :shell, :inline => "ssh -o StrictHostKeyChecking=no pcmk-2 'modprobe drbd'"
    pcmk1.vm.provision :shell, :inline => "ssh -o StrictHostKeyChecking=no pcmk-2 'drbdadm up wwwdata'"
    pcmk1.vm.provision :shell, :inline => "ssh -o StrictHostKeyChecking=no pcmk-2 'cat /proc/drbd'"
    pcmk1.vm.provision :shell, :inline => "drbdadm create-md wwwdata"
    pcmk1.vm.provision :shell, :inline => "modprobe drbd"
    pcmk1.vm.provision :shell, :inline => "drbdadm up wwwdata"
    pcmk1.vm.provision :shell, :inline => "cat /proc/drbd"
    pcmk1.vm.provision :shell, :inline => "drbdadm primary --force wwwdata"
    pcmk1.vm.provision :shell, :inline => "cat /proc/drbd"
    pcmk1.vm.provision :shell, :inline => "mkfs.ext4 /dev/drbd1"
    pcmk1.vm.provision :shell, :inline => "mount /dev/drbd1 /mnt"
    pcmk1.vm.provision :shell, :inline => $mnt_index_html
    pcmk1.vm.provision :shell, :inline => "umount /dev/drbd1"
    pcmk1.vm.provision :shell, :inline => "pcs cluster cib drbd_cfg"
    pcmk1.vm.provision :shell, :inline => "pcs -f drbd_cfg resource create WebData ocf:linbit:drbd drbd_resource=wwwdata op monitor interval=60s"
    pcmk1.vm.provision :shell, :inline => "pcs -f drbd_cfg resource master WebDataClone WebData master-max=1 master-node-max=1 clone-max=2 clone-node-max=1 notify=true"
    pcmk1.vm.provision :shell, :inline => "pcs -f drbd_cfg resource show"
    pcmk1.vm.provision :shell, :inline => "pcs cluster cib-push drbd_cfg"
    pcmk1.vm.provision :shell, :inline => "pcs status"
    pcmk1.vm.provision :shell, :inline => "echo drbd > /etc/modules-load.d/drbd.conf"
    pcmk1.vm.provision :shell, :inline => "pcs cluster cib fs_cfg"
    pcmk1.vm.provision :shell, :inline => "pcs -f fs_cfg resource create WebFS Filesystem device='/dev/drbd1' directory='/var/www/html' fstype='ext4'"
    pcmk1.vm.provision :shell, :inline => "pcs -f fs_cfg constraint colocation add WebFS with WebDataClone INFINITY with-rsc-role=Master"
    pcmk1.vm.provision :shell, :inline => "pcs -f fs_cfg constraint order promote WebDataClone then start WebFS"
    pcmk1.vm.provision :shell, :inline => "pcs -f fs_cfg constraint colocation add WebSite with WebFS INFINITY"
    pcmk1.vm.provision :shell, :inline => "pcs -f fs_cfg constraint order WebFS then WebSite"
    pcmk1.vm.provision :shell, :inline => "pcs -f fs_cfg constraint"
    pcmk1.vm.provision :shell, :inline => "pcs -f fs_cfg resource show"
    pcmk1.vm.provision :shell, :inline => "pcs cluster cib-push fs_cfg"
    pcmk1.vm.provision :shell, :inline => "pcs status"
    pcmk1.vm.provision :shell, :inline => "pcs cluster standby pcmk-1"
    pcmk1.vm.provision :shell, :inline => "pcs status"
    pcmk1.vm.provision :shell, :inline => "pcs cluster unstandby pcmk-1"
    pcmk1.vm.provision :shell, :inline => "pcs status"
    # Fencing missing!
    # kernel-modules-extra for the running kernel
    pcmk1.vm.provision :shell, :inline => "yum install -y kernel-modules-extra-`uname -r` kernel-headers-`uname -r`"
    pcmk1.vm.provision :shell, :inline => "yum install -y gfs2-utils dlm"
    pcmk1.vm.provision :shell, :inline => "pcs cluster cib dlm_cfg"
    pcmk1.vm.provision :shell, :inline => "pcs -f dlm_cfg resource create dlm ocf:pacemaker:controld op monitor interval=60s"
    pcmk1.vm.provision :shell, :inline => "pcs -f dlm_cfg resource clone dlm clone-max=2 clone-node-max=1"
    pcmk1.vm.provision :shell, :inline => "pcs -f dlm_cfg resource show"
    pcmk1.vm.provision :shell, :inline => "pcs cluster cib-push dlm_cfg"
    pcmk1.vm.provision :shell, :inline => "pcs status"
  end
end
