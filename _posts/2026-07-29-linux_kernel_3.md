---
title: "Linux Kernel: Build with Busybox for debugging"
date: 2026-07-29 10:00:00 +0900
categories: [Kernel, Pwnable]
tags: [linux, kernel]
---

참고1: [https://jeongzero.oopy.io/73084e52-54fa-43e2-986b-072ee2a4f80d](https://jeongzero.oopy.io/73084e52-54fa-43e2-986b-072ee2a4f80d)  
참고2: [https://www.minzkn.com/linuxkernel/pages/gdb-kernel.html](https://www.minzkn.com/linuxkernel/pages/gdb-kernel.html)
# 1. Build Busybox
```sh
$ wget https://busybox.net/downloads/busybox-1.38.0.tar.bz2
$ tar -xvf busybox-1.31.0.tar.bz2
$ cd busybox-1.38.0/
$ make defconfig
$ make menuconfig
```

1. menuconfig -> settings -> Build static binary (no shared libs) 클릭
![Build static binary option](/assets/img/linux_kernel_3/busybox_static_binary.png)

2. menuconfig -> Networking Utilities -> tc 해제  
![tc option](/assets/img/linux_kernel_3/busybox_tc.png)

```sh
$ mkdir _install
$ make CONFIG_PREFIX=_install
$ make -j$(nproc)
$ make install
```
만약 위의 명령어 과정에서 오류 발생 시

```sh
$ make clean
$ rm -rf _install
$ make defconfig
$ sed -i 's/CONFIG_TC=y/# CONFIG_TC is not set/' .config
$ sed -i 's/CONFIG_FEATURE_TC_INGRESS=y/# CONFIG_FEATURE_TC_INGRESS is not set/' .config
$ sed -i 's/# CONFIG_STATIC is not set/CONFIG_STATIC=y/' .config
$ make -j$(nproc)
$ make install
```

성공 시 아래의 파일이 생기게 된다.  
![busybox_result](/assets/img/linux_kernel_3/busybox_build_result.png)

이후에는
<details>
<summary>chardev.c</summary>

<div markdown="1">
```c
#include <linux/init.h>
#include <linux/module.h>
#include <linux/types.h>
#include <linux/kernel.h>
#include <linux/fs.h>
#include <linux/cdev.h>
#include <linux/sched.h>
#include <linux/device.h>
#include <linux/slab.h>
#include <asm/current.h>
#include <linux/uaccess.h>
#include <linux/string.h>

MODULE_LICENSE("Dual BSD/GPL");

#define DRIVER_NAME "chardev"
#define BUFFER_SIZE 256

static const unsigned int MINOR_BASE = 0;
static const unsigned int MINOR_NUM  = 2;
static unsigned int chardev_major;
static struct cdev chardev_cdev;
static struct class *chardev_class = NULL;

static int     chardev_open(struct inode *, struct file *);
static int     chardev_release(struct inode *, struct file *);
static ssize_t chardev_read(struct file *, char __user *, size_t, loff_t *);
static ssize_t chardev_write(struct file *, const char __user *, size_t, loff_t *);

struct file_operations chardev_fops = {
    .open    = chardev_open,
    .release = chardev_release,
    .read    = chardev_read,
    .write   = chardev_write,
};

struct data {
    unsigned char buffer[BUFFER_SIZE];
};

static int chardev_init(void)
{
    int alloc_ret = 0;
    int cdev_err = 0;
    int minor;
    dev_t dev;

    printk(KERN_INFO "The chardev_init() function has been called.\n");

    alloc_ret = alloc_chrdev_region(&dev, MINOR_BASE, MINOR_NUM, DRIVER_NAME);
    if (alloc_ret != 0) {
        printk(KERN_ERR "alloc_chrdev_region = %d\n", alloc_ret);
        return -1;
    }
    chardev_major = MAJOR(dev);
    dev = MKDEV(chardev_major, MINOR_BASE);

    cdev_init(&chardev_cdev, &chardev_fops);
    chardev_cdev.owner = THIS_MODULE;

    cdev_err = cdev_add(&chardev_cdev, dev, MINOR_NUM);
    if (cdev_err != 0) {
        printk(KERN_ERR "cdev_add = %d\n", cdev_err);
        unregister_chrdev_region(dev, MINOR_NUM);
        return -1;
    }

    /* Linux 6.6.14 호환 */
    chardev_class = class_create("chardev");
    if (IS_ERR(chardev_class)) {
        printk(KERN_ERR "class_create failed\n");
        cdev_del(&chardev_cdev);
        unregister_chrdev_region(dev, MINOR_NUM);
        return -1;
    }

    for (minor = MINOR_BASE; minor < MINOR_BASE + MINOR_NUM; minor++) {
        device_create(chardev_class, NULL, MKDEV(chardev_major, minor), NULL, "chardev%d", minor);
    }

    return 0;
}

static void chardev_exit(void)
{
    int minor;
    dev_t dev = MKDEV(chardev_major, MINOR_BASE);

    printk(KERN_INFO "The chardev_exit() function has been called.\n");

    for (minor = MINOR_BASE; minor < MINOR_BASE + MINOR_NUM; minor++) {
        device_destroy(chardev_class, MKDEV(chardev_major, minor));
    }

    class_destroy(chardev_class);
    cdev_del(&chardev_cdev);
    unregister_chrdev_region(dev, MINOR_NUM);
}

static int chardev_open(struct inode *inode, struct file *file)
{
    char *str = "helloworld";

    struct data *p = kmalloc(sizeof(struct data), GFP_KERNEL);

    printk(KERN_INFO "The chardev_open() function has been called.\n");

    if (p == NULL) {
        printk(KERN_ERR "kmalloc - Null\n");
        return -ENOMEM;
    }

    /* Linux 6.x 호환 */
    strscpy(p->buffer, str, sizeof(p->buffer));

    file->private_data = p;
    return 0;
}

static int chardev_release(struct inode *inode, struct file *file)
{
    printk(KERN_INFO "The chardev_release() function has been called.\n");
    if (file->private_data) {
        kfree(file->private_data);
        file->private_data = NULL;
    }
    return 0;
}

static ssize_t chardev_write(struct file *filp, const char __user *buf, size_t count, loff_t *f_pos)
{
    struct data *p = filp->private_data;

    printk(KERN_INFO "The chardev_write() function has been called.\n");
    printk(KERN_INFO "Before copy_from_user : %p, %s\n", p->buffer, p->buffer);

    if (copy_from_user(p->buffer, buf, count) != 0) {
        return -EFAULT;
    }

    printk(KERN_INFO "After copy_from_user : %p, %s\n", p->buffer, p->buffer);
    return count;
}

static ssize_t chardev_read(struct file *filp, char __user *buf, size_t count, loff_t *f_pos)
{
    struct data *p = filp->private_data;

    printk(KERN_INFO "The chardev_read() function has been called.\n");

    if (count > BUFFER_SIZE) {
        count = BUFFER_SIZE;
    }

    if (copy_to_user(buf, p->buffer, count) != 0) {
        return -EFAULT;
    }

    return count;
}

module_init(chardev_init);
module_exit(chardev_exit);
```
</div>
</details>

<details>
<summary>Makefile</summary>

<div>
``` makefile
obj-m += chardev.o

KDIR := /home/sy46/kernel_lab/linux-6.6.14

all:
        make -C $(KDIR) M=$(shell pwd) modules

clean:
        make -C $(KDIR) M=$(shell pwd) clean
```
</div>
</details>

```sh
$ cd ~/busybox-1.38.0/_install
# 위의 chardev.c, Makefile 생성
$ mkdir -p dev proc sys etc root
$ make
$ cp chardev.ko /home/sy46/busybox-1.38.0/_install/root/
$ cat << 'EOF' > init
#!/bin/sh
mount -t proc none /proc
mount -t sysfs none /sys
echo "========================================"
echo "  Success! Welcome to Kernel Lab Shell  "
echo "========================================"
exec /bin/sh
EOF

$ chmod +x init
$ find . -print0 | cpio --null -ov --format=newc | gzip -9 > ./initramfs.cpio.gz
```

# 2. Build Kernel
```sh
$ mkdir kernel_lab
$ wget https://cdn.kernel.org/pub/linux/kernel/v6.x/linux-6.6.14.tar.xz
$ tar -xf linux-6.6.14.tar.xz
$ cd linux-6.6.14
$ make olddefconfig
$ make defconfig
$ make -j$(nproc)
```
KGDB를 설정해주고, 다른 필요한 defconfig들을 설정해준다.  
![kgdb option](/assets/img/linux_kernel_3/kernel_kgdb.png)


# 3. Start with QEMU
```sh
$ qemu-system-x86_64 \
    -m 2G \
    -kernel /home/sy46/kernel_lab/linux-6.6.14/arch/x86/boot/bzImage \
    -initrd /home/sy46/busybox-1.38.0/_install/initramfs.cpio.gz \
    -append "console=tty0 nokaslr panic=0" \
    -s

```
첫 번째 터미널에서 qemu를 실행한다.
성공 화면은 아래와 같다.
![qemu_booting](/assets/img/linux_kernel_3/qemu_booting.png)


두 번째 터미널에서 remote로 접속한다.
```sh
# pnwdbg
# $ gdb-multiarch -q /home/sy46/kernel_lab/linux-6.6.14/vmlinux

# original gdb
# $ gdb -nh ~/kernel_lab/linux-6.6.14/vmlinux

# gef
$ gdb ~/kernel_lab/linux-6.6.14/vmlinux
(gdb) target remote :1234
(gdb) continue
```

이후 insmod를 통해 생성한 커널 모듈을 삽입한다.
![insmod_chardev](/assets/img/linux_kernel_3/insmod_chardev.png)  
그런데 만약 /dev 경로에 console밖에 없다면
```sh
$ mount -t devtmpfs devtmpfs /dev
```
위를 입력해야 한다.

다음으로는 chardev.ko가 매핑된 주소를 읽어야 한다.  
```sh
$ cat /sys/module/chardev/sections/.text
```
![chardev_mapping](/assets/img/linux_kernel_3/chardev_mapping.png)  

해당 주소를 바탕으로 심볼을 등록한다.  
```sh
$ add-symbol-file /home/sy46/busybox-1.38.0/_install/chardev.ko 0xffffffffc0000000
# y
$ hb chardev_open
$ hb chardev_write
```
![add_symbol](/assets/img/linux_kernel_3/add_symbol.png)

성공 시 아래처럼 bp가 걸리게 된다.
![chardev_break](/assets/img/linux_kernel_3/chardev_break.png)  
자세히 보고싶다면 `list`명령어를 입력하면 되고,

만약 pwndbg로 실행했다면 `context` 명령어를 통해 아래 화면을 볼 수 있다.
![kernel_pwndbg](/assets/img/linux_kernel_3/kernel_pwndbg.png)

만약 gef로 실행한다면  
![gef](/assets/img/linux_kernel_3/gef.png)  
위처럼 볼 수 있다.  
여기서 gef가 커널 디버깅에 가장 잘 어울린다고 한다.  

참고로 위의 qemu 실행을 쉘 스크립트로 구현할 수 있는데,  
```sh
qemu-system-x86_64 \
-m 4G -smp 4,cores=4,threads=1 \
-kernel /home/sy46/kernel_lab/linux-6.6.14/arch/x86/boot/bzImage \
-initrd  /home/sy46/busybox-1.38.0/_install/initramfs.cpio \
-append "root=/dev/ram rw console=ttyS0 oops=panic panic=1 quiet nopti" \
-netdev user,id=t0, -device e1000,netdev=t0,id=nic0 \
-nographic  \
-cpu host \
-enable-kvm \
-s \
```
위와 비슷하게 구현하면 된다고 한다.  

또한, init 파일도 수정하면
```sh
#!/bin/sh

mount -t proc none /proc
mount -t sysfs none /sys
mount -t devtmpfs devtmpfs /dev
exec 0</dev/console
exec 1>/dev/console
exec 2>/dev/console

echo "7 4 1 7" > /proc/sys/kernel/printk

cp /proc/kallsyms /tmp/kallsyms

setsid cttyhack setuidgid 1000 sh

umount /proc
umount /sys
poweroff -d 0  -f
```
위를 통해 init process 실행 시 초기화가 진행된다고 한다.  
qemu 쉘 스크립트와 init 파일은 나중에 수정해야겠다.
