Only usable on systems with minimum 1G RAM
Install Debian jessie on your server
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
	set hwmac = concat (
	suffix (concat ("0", binary-to-ascii (16, 8, "", substring(hardware,1,1))),2), ":",
	suffix (concat ("0", binary-to-ascii (16, 8, "", substring(hardware,2,1))),2), ":",
	suffix (concat ("0", binary-to-ascii (16, 8, "", substring(hardware,3,1))),2), ":",
	suffix (concat ("0", binary-to-ascii (16, 8, "", substring(hardware,4,1))),2), ":",
	suffix (concat ("0", binary-to-ascii (16, 8, "", substring(hardware,5,1))),2), ":",
	suffix (concat ("0", binary-to-ascii (16, 8, "", substring(hardware,6,1))),2)
	);

      filename = concat( "http://$FQDN/", hwmac );
   }

# Actual build starts here

DIR=/srv/live-build-x2go-`date +"%Y%m%d%H%M%S"`
mkdir $DIR
cd $DIR
lb config --chroot-filesystem squashfs --apt-indices none --cache-packages false --config https://github.com/LinuxHaus/live-build-x2go.git::feature/bionic --archive-areas "main contrib non-free" --apt-recommends false --firmware-binary true --backports false --win32-loader false --security true  --initsystem systemd -d bionic -b netboot -a amd64 -k amd64 --linux-packages linux-image --bootappend-live vconsole.keymap=de-latin1-nodeadkeys log_buf_len=1M quickreboot silent splash lang=de locales=de_DE.UTF-8 keyboard-layouts=de consoleblank=0 quiet kernel.sysrq=1 keep_bootcon sysrq_always_enabled toram live-config live-config.timezone=Europe/Berlin boot=live
lb build
cd `dirname $DIR`
ln -s `basename $DIR` lb-x2go
cd /var/www/html
ln -s /srv/lb-x2go/tftpboot/live/initrd.img .
ln -s /srv/lb-x2go/tftpboot/live/vmlinuz .
ln -s /srv/lb-x2go/binary/live/filesystem.squashfs .
YOURHW=myhw
cat <<EOF>$YOURHW
#!ipxe
dhcp
kernel http://$FQDN/vmlinuz boot=live components fetch=http://$IP_OF_FQDN/filesystem.squashfs noswap vconsole.keymap=de-latin1-nodeadkeys log_buf_len=1M quickreboot silent splash quiet lang=de locales=de_DE.UTF-8 keyboard-layouts=de consoleblank=0 quiet kernel.sysrq=1 keep_bootcon sysrq_always_enabled initrd=initrd.img rootwait=120 live-config
initrd http://$FQDN/initrd.img
boot
EOF
ln -s myhw $MAC_OF_YOURCLIENT
