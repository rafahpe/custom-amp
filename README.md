# custom-amp

This repo contains instructions to build a customized installation image for Aruba Airwave 8.X

## Motivation

The installation image of Airwave features a "stable" release of CentOS (6.9 as of writing this) that does not include drivers for many recent hardware. Besides, as of 8.2.5, Airwave now comes in the form of a closed appliance with no root shell access, so you cannot install drivers afterwards without TAC support.

So, if you want to install Airwave in your newest and shiniest box, you may find out that you can't because, for instance, the network card is not recognized and the installation won't proceed.

The instructions in this repo help you build a customized installation ISO for Airwave that includes the packages that you need.

## DISCLAIMER

THIS IS A PERSONAL HOW-TO. THIS PAGE IS NOT AFFILIATED IN ANY MANNER TO HPE OR ARUBA. THIS IS NOT A SUPPORTED GUIDE, NOR DEPLOYING AIRWAVE IN UNSUPPORTED HARDWARE IS ENCOURAGED BY ANY MEANS, BE ME OR HPE ARUBA. AND OF COURSE, SUCH AN INSTALLATION WOULD NOT BE SUPPORTED BY THE VENDOR. REDISTRIBUTION OF A MODIFIED AIRWAVE IMAGE IS OF COURSE NOT ALLOWED, AS STATED BY THE AIRWAVE LICENSE.

## Step 1 - Install Airwave

To package a customized ISO, you will first need a development environment to work in. The best option is to install Airwave in a virtual machine, and use that installation as your baseline.

I used KVM and virt-manager to install an Airwave VM. Deploying VMs in separate hypervisors is out of the scope of this document. The resources I allocated for my Airwave VM are:

- 2 vCPUs
- 4 GB RAM
- 80 GB HDD (virtio)
- 1 NIC (virtio)

After installation completed, I booted the machine from a CentOS 7 install CD and started a terminal session, to overwirte the root password and gain root access to the box:

```bash
# Activate the lvm volumes
vgchange -a y VolGroup00
# Mount the filesystem
mount /dev/VolGroup00/LogVol00 /mnt
# Change the root password
chroot /mnt
passwd root

# Change the root shell
usermod -s /bin/bash root
# Reboot
reboot
```

After reboot, you can login with root user and whatever password you configured for it.

## Step 2 - Prepare your kickstart folder

The instructions for this part are borrowed from http://www.smorgasbork.com/2014/07/16/building-a-custom-centos-7-kickstart-disc-part-1/.

Copy the AMP iso to your CentOS environment, using scp or winscp:

```bash
scp AMP-<whatever-version-ypu-have>.iso root@amp.server.ip.addr:/root
```

Log into your CentOS environment as root and type the following commands:

```bash
export BUILD_DIR=/root/kickstart_build
mkdir -p "${BUILD_DIR}"
```

Mount your ISO image inside the CentOS environment:

```bash
mkdir -p /tmp/amp
mount -o loop -t iso9660 /root/AMP*.iso /tmp/amp
```

Copy bootstrap files from your ISO to the kickstart folders:

```bash
export LC_ALL=C
cp /tmp/amp/.discinfo "${BUILD_DIR}/.discinfo"
cp -pr /tmp/amp/* "${BUILD_DIR}/"
rm -rf "${BUILD_DIR}/repodata"
mkdir -p "${BUILD_DIR}/repodata"
cp /tmp/amp/repodata/*-comps.xml.gz "${BUILD_DIR}/comps.xml.gz"
gunzip "${BUILD_DIR}/comps.xml.gz"
cp -f /root/anaconda-ks.cfg "${BUILD_DIR}/ks.cfg"
```

## Step 3 - Customize your ks.cfg

There are a few things you may want to do to your kickstart file.

### Remove network dependencies

If you are building this iso image to install a network driver, you probably don't want the installer to halt because of not finding an eth0 interface. You better remove the following line from `${BUILD_DIR}/ks/ks.cfg`:

```
network --onboot yes --device eth0 --bootproto dhcp
```

Replace with:

```
network  --device=lo --hostname=localhost.localdomain
```

You also want to **remove** this line in your ks.cfg:

```
repo --name="CentOS"  --baseurl=cdrom:sr0 --cost=100
```

### Replace root password

You probably want to generate a known password for the root user. As of 8.2.6.1, Airwave uses MD5 hashing for shadow passwords, so you can generate a password with the following command:

(**Replace "My Password" with your value**)

```bash
export MYSALT=`head /dev/urandom | tr -dc A-Za-z0-9 | head -c 8`
python -c "import crypt; print(crypt.crypt('My Password', '\$1\$$MYSALT'))"

# You get
>> $1$z2Rsrwux$WGQYUEU8..42drxh6oJpp0
```

Replace the string you got in the following line of your `${BUILD_DIR}/ks/ks.cfg` file:

```
rootpw  --iscrypted $1$z2Rsrwux$WGQYUEU8..42drxh6oJpp0
```

### Replace your bootloader drive

Probably, your definitive hardware won't have a virtio disk like the virtual devel environment has. You should replace the "bootloader" line in your ks.cfg file to specifiy the proper boot disk:

```bash
# Replace vda with the proper disk name! usually sda por SCSI / SATA disks
bootloader --location=mbr --driveorder=vda --append="ide=nodma crashkernel=auto rhgb quiet"
```

Also, uncomment the commands that format the disk. Beware! **the sample below is for AW 8.2.6.1, your version may have a different disk geometry. Check the commented lines in ks.cfg**.:

```bash
clearpart --drives=sda --all --initlabel

part /boot/efi --fstype=efi --fsoptions="defaults,uid=0,gid=0,umask=0077,shortname=winnt" --grow --maxsize=200 --size=20
part /boot --fstype=ext4 --size=100
part pv.253003 --grow --size=5112

volgroup VolGroup00 --pesize=32768 pv.253003
logvol swap --name=LogVol01 --vgname=VolGroup00 --size=4096
logvol / --fstype=ext4 --name=LogVol00 --vgname=VolGroup00 --grow --size=1024
```

## Step 4 - Add packages

You can add rpm packages to the `${BUILD_DIR}/Packages` folder of your setup. You have to add **all dependencies** of the package, along with the package itself.

For instance, to install AirWave in a recent Dell server, I downloaded the driver for the intel X710 network card from `https://www.dell.com/support/home/es/es/esbsdt1/Drivers/DriversDetails?driverId=M48PP`, and copied the RPM `kmod-i40e-2.1.32-1.el6.x86_64.rpm` into the `${BUILD_DIR}/Packages` folder.
 
To check if your package set is complete, and includes all dependencies, you can run the following commands:

```bash
cd "${BUILD_DIR}/Packages"
mkdir /tmp/testdb

rpm --initdb --dbpath /tmp/testdb
rpm --test --dbpath /tmp/testdb -Uvh *.rpm
```

You have to keep adding dependencies until you get no missing dependencies errors. Finally, add your packages to the end of the `%packages` section of the ks.cfg file:

```bash
%packages
@Base
@Core
MAKEDEV
acl
# Omitted for brevity
# ... 
# Now, your packages:
kmod-i40e

%end
```

## Post-install

You can also modify the post-install settings in the ks.cfg. You will probably want to keep the bash shell for the root user, instead of replacing it by the airwave command-line menu. Locate and comment out the following line in `ks.cfg`:

```
/usr/sbin/usermod -s /root/git/mercury/install/amp-install root
```

## Build your image

```bash
yum install createrepo genisoimage isomd5sum
```

If your CentOS version is CentOS 6.X (AirWave 8.2.6 is Centos 6.9), you create the repo using the following commands:

```bash
cd "${BUILD_DIR}"
declare -x discinfo=`head -1 .discinfo`
createrepo -u "media://$discinfo" -g "${BUILD_DIR}/comps.xml" .
```

If your CentOS is 7.X instead, you use these commands:

```bash
cd "${BUILD_DIR}"
createrepo -g "${BUILD_DIR}/comps.xml" .
```

Now you can finally build the ISO! To be able to boot in EFI servers, you will need xorriso instead of mkisofs (see https://askubuntu.com/questions/625286/how-to-create-uefi-bootable-iso). The safest bet is to **copy the boot sector from the original ISO file** and tell xorriso to use it:

```bash
cd "${BUILD_DIR}"
dd if=<your original AMP iso> bs=512 count=1 of=isohdpfx.bin
cd ..
xorriso -as mkisofs \
  -isohybrid-mbr "${BUILD_DIR}/isohdpfx.bin" \
  -c isolinux/boot.cat \
  -b isolinux/isolinux.bin \
  -no-emul-boot \
  -boot-load-size 4 \
  -boot-info-table \
  -eltorito-alt-boot \
  -e boot/grub/efi.img \
  -no-emul-boot \
  -isohybrid-gpt-basdat \
  -o custom-amp.iso \
  "${BUILD_DIR}"
#mkisofs -o ../custom-amp.iso -b isolinux/isolinux.bin -c boot.cat \
#   -no-emul-boot -boot-load-size 4 -boot-info-table -R -J -v -T .
/usr/bin/implantisomd5 ../custom-amp.iso
```

## Install AirWave

To install airwave with your new custom ISO, boot from the image and type the following in the boot menu, when prompted:

- Centos 6:

```
linux ks=cdrom:/ks.cfg
```

- Centos 7:

```
linux inst.ks=cdrom:/dev/cdrom:/ks/ks.cfg
```

