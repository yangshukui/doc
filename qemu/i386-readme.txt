i386 busybox-1.9.2
qemu -kernel bzImage -hda rootfs.img -append "root=/dev/hda init=/bin/ash"

dd if=/dev/zero of=rootfs.img bs=1M count=10 //������rootfs.img����С10M
mke2fs -F -v initrd.img //mkfs.ext4 rootfs.img //��rootfs.img������ext4�ļ�ϵͳ
mkdir rootfs
mount -o loop initrd.img initrd
cd rootfs
cp -a busybox/build/dir/_install/* ./
mkdir -v etc dev
cp -a busybox-1.19.4/examples/bootfloppy/etc/* ./etc
cp -R /dev/null ./dev/
cp -R /dev/console ./dev/
�� .../arm-none-linux-gnueabi/libc/usr/include/linux/netfilter.h �Ŀ�ͷ
���ȱ�ٵ�ͷ�ļ���
#include <netinet/in.h>

make install CONFIG_PREFIX=<path to rootfs>

linux-2.6.14
CFLAGS���������-fno-stack-protector gcc-4.1.3

make oldconfig
make menuconfig

qemu-system -kernel bzImage -hda rootfs.img -append "root=/dev/sda init=/bin/ash"