Only usable on systems with minimum 1G RAM
Install Debian buster on your server
become root on the server
apt update
apt install ipxe atftpd atftp git-core xinetd apache2 dnsutils live-build live-config-doc live-manual-html live-boot-doc

# tftp/pxe config
cat <<EOF>/etc/default/atftpd 
USE_INETD=false
OPTIONS="--daemon --port 69 --retry-timeout 5 --no-multicast --maxthread 100 --verbose=5 /srv/tftp"
EOF
mkdir -p /srv/tftp/{bios,uefi}
ln -s /usr/lib/ipxe/undionly.kpxe /srv/tftp/bios/
ln -s /boot/ipxe.efi /srv/tftp/uefi/
FQDN=demo.x2go.org
IP_OF_FQDN=`dig $FQDN +short`

# just a part of dhcpd.conf from isc-dhcp-server package:
group {
   if substring ( option vendor-class-identifier , 19,1 ) = "0" {
           filename "bios/undionly.kpxe";
   }
   else if substring ( option vendor-class-identifier , 19,1 ) = "7" {
           filename "uefi/ipxe.efi";
   }
   else {  
           log (info, concat ( "Unhandled vendor class Arch: ", substring ( option vendor-class-identifier , 19,1 )));
   }
   if exists user-class and option user-class = "iPXE" {
      filename = "http://$FQDN/myhw";
   }
   host examplehost { hardware ethernet 0:0:c0:5d:bd:95; }

}

# Actual build starts here

DIR=/srv/live-build-x2go-`date +"%Y%m%d%H%M%S"`
mkdir $DIR
cd $DIR
lb config --debootstrap-options "--exclude=nano,iptables,dmidecode,info" --apt-source-archives false --cache false --apt-http-proxy http://approx:3142 --chroot-filesystem squashfs --apt-indices none --cache-packages false --config git://code.x2go.org/live-build-x2go.git::master --archive-areas "main contrib" --apt-recommends false --firmware-chroot false --firmware-binary false --updates true --backports false --win32-loader false --loadlin false --security true  --initsystem systemd  -b netboot -a amd64 -k amd64 --linux-packages linux-image --bootappend-live vconsole.keymap=de-latin1-nodeadkeys log_buf_len=1M quickreboot silent splash lang=de locales=de_DE.UTF-8 keyboard-layouts=de consoleblank=0 quiet kernel.sysrq=1 keep_bootcon sysrq_always_enabled toram live-config live-config.timezone=Europe/Berlin 
lb build
cd `dirname $DIR`
ln -s `basename $DIR` lb-x2go
cd /var/www/html
ln -s /srv/lb-x2go/binary/live .
cat <<EOF>myhw
#!ipxe
dhcp
kernel http://$FQDN/live/vmlinuz boot=live components fetch=http://$IP_OF_FQDN/live/filesystem.squashfs noswap vconsole.keymap=de-latin1-nodeadkeys log_buf_len=1M quickreboot silent splash toram lang=de locales=de_DE.UTF-8 keyboard-layouts=de consoleblank=0 quiet kernel.sysrq=1 keep_bootcon sysrq_always_enabled initrd=initrd.img rootwait=120 live-config
initrd http://$FQDN/live/initrd.img
boot
EOF
