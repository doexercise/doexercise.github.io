# Marvell armada 7040 SoC 의 BSP 작업 기록

> update : Nov 9 2018  
> author : me

&nbsp;

---

## 작업환경

- OS  : ubuntu 18.04, VirtualBox VM  
- SDK : 2018 September (18.09.4) from Marvell  
- Required packages : libssl-dev device-tree-compiler swig libpython-dev bison flex libncurses-dev curl git

```shell
root@buildsrv:~# apt-get install libssl-dev device-tree-compiler swig libpython-dev bison flex libncurses-dev curl
```

기본적인 작업내용은 `Software -> 2018 September (18.9.4) -> Documentation -> a80x0-a70x0-a39xx-a37xx-a38x-devel-18.09.4-docs.zip` 에 들어있다. 이 파일을 압축해제하고 index.html 을 실행하라.

&nbsp;

---

## Cross Compiler 설치  

1. 빌드서버에 파일 복사
    - 파일 위치 : Software -> 2018 September (18.9.4) -> SDK Components -> Toolchain -> gcc-7.3.1-linaro-2018.05
    - 위 위치에서 압축 파일을 풀어 `gcc-linaro-7.3.1-2018.05-x86_64_aarch64-linux-gnu.tar.xz` 파일을 빌드시스템에 복사

2. 압축해제 및 PATH 설정
    ```shell
    root@buildsrv:~# xz -d gcc-linaro-7.3.1-2018.05-x86_64_aarch64-linux-gnu.tar.xz
    root@buildsrv:~# tar xf gcc-linaro-7.3.1-2018.05-x86_64_aarch64-linux-gnu.tar
    root@buildsrv:~# ln -s gcc-linaro-7.3.1-2018.05-x86_64_aarch64-linux-gnu toolchain
    root@buildsrv:~# export CROSS_COMPILE=/root/toolchain/bin/aarch64-linux-gnu-
    ```

## U-Boot  

1. Marvell 에서 제공하는 소스 파일을 사용하는 방법
    - 파일 위치 : Software -> 2018 September (18.9.4) -> SDK Components -> Firmware & Boot Loaders -> U-Boot 에서 `bootloaders-sources-standalone-18.09.4.zip` 과 `bootloaders-git-standalone-18.09.4.zip` 을 다운로드 하고 빌드 서버로 복사
    - 파일을 복사/압축해제 하고 패치한다.
    ```shell
    root@buildsrv:~# mkdir -p uboot/src
    root@buildsrv:~# mkdir -p uboot/patch
    root@buildsrv:~# unzip -d uboot/src bootloaders-sources-standalone-18.09.4.zip
    root@buildsrv:~# unzip -d uboot/patch bootloaders-git-standalone-18.09.4.zip
    root@buildsrv:~# cd uboot/src
    root@buildsrv:~/uboot/src# git init
    root@buildsrv:~/uboot/src# git add .
    root@buildsrv:~/uboot/src# git commit -m "init"
    root@buildsrv:~/uboot/src# git am ../patch/*.patch
    ```

2. git 을 이용하는 방법
    - Marvell 에서 제공하는 패치 파일을 복사/압축해제한다. 패치 파일 위치는 `/root/uboot-patch` 가 될 것이다.
    ```shell
    root@buildsrv:~# unzip -d uboot-patch bootloaders-git-standalone-18.09.4.zip
    ```
    - git 을 이용하여 소스를 다운로드 한다
    ```shell
    root@buildsrv:~# git clone git://git.denx.de/u-boot.git
    root@buildsrv:~# mv u-boot u-boot-2018.03-18.09.4
    root@buildsrv:~# cd u-boot-2018.03-18.09-4
    root@buildsrv:~# git checkout v2018.03 -b marvell-18.09.4
    ```
    - 패치한다
    ```shell
    root@buildsrv:~# cd u-boot-2018.03-18.09.4
    root@buildsrv:~/u-boot-2018.03-18.09.4# git am -3 /root/uboot-patch/*.patch
    ```
** Default Environment **

Marvell>> pri
baudrate=115200
bootargs=console=ttyS0,115200 /dev/nfs ip=192.168.3.29:192.168.3.57:192.168.3.254:255.255.255.0:marvell:eth0:none nfsroot=192.168.3.57:/root/service/nfs/ktgsap2
bootcmd=run get_images; run set_bootargs; booti $kernel_addr $ramfs_addr $fdt_addr
bootdelay=2
console=console=ttyS0,115200
eth1addr=00:51:82:11:22:01
eth2addr=00:51:82:11:22:02
eth3addr=00:51:82:11:22:03
ethact=mvpp2-1
ethaddr=00:51:82:11:22:00
ethprime=eth1
fdt_addr=0x4f00000
fdt_high=0xffffffffffffffff
fdt_name=gsap-v2/armada-7040-db-GSAP.dtb
fdtcontroladdr=7f6decb8
fileaddr=5000000
filesize=c35200
gatewayip=192.168.3.254
get_images=tftpboot $kernel_addr $image_name; tftpboot $fdt_addr $fdt_name; run get_ramfs
get_ramfs=if test "${ramfs_name}" != "-"; then setenv ramfs_addr 0x8000000; tftpboot $ramfs_addr $ramfs_name; else setenv ramfs_addr -;fi
hostname=marvell
image_name=Image
initrd_addr=0xa00000
initrd_size=0x2000000
ipaddr=192.168.3.29
kernel_addr=0x5000000
loadaddr=0x5000000
netdev=eth0
netmask=255.255.255.0
nfsboot=set rootpath /root/service/nfs/ktgsap2; setenv bootargs console=ttyS0,115200n8 root=/dev/nfs libata.force=noncq rw nfsroot=192.168.3.57:/root/service/nfs/ktgsap2 ip=192.168.3.29:192.168.3.57:192.168.3.254:255.255.255.0::eth1:off; tftp ${loadaddr} gsap-v2/Image; tftp ${fdt_addr}  gsap-v2/armada-7040-db-GSAP.dtb; booti ${loadaddr} - ${fdt_addr}
ramfs_addr=-
ramfs_name=-
root=/dev/nfs
rootpath=/root/service/nfs/ktgsap2
sataboot=run sataboot_load; run sataboot_args; booti 0x5000000 - 0x4f00000
sataboot_args=setenv bootargs console=ttyS0,115200 root=/dev/sda2 rw
sataboot_load=scsi scan; ext2load scsi 0 0x5000000 Image; ext4load scsi 0 0x4f00000 armada-7040-db-GSAP.dtb
serverip=192.168.3.57
set_bootargs=setenv bootargs $console $root ip=$ipaddr:$serverip:$gatewayip:$netmask:$hostname:$netdev:none nfsroot=$serverip:$rootpath $extra_params
stderr=serial@512000
stdin=serial@512000
stdout=serial@512000

Environment size: 2005/65532 bytes

=======================================================================================================

** for NFS boot

setenv bootargs console=ttyS0,115200 root=/dev/nfs libata.force=noncq rw nfsroot=192.168.3.57:/nfsroot,vers=4,tcp ip=192.168.3.29:192.168.3.57:192.168.3.254:255.255.255.0:marvell:eth1:off; tftp ${loadaddr} gsap-v2/Image; tftp ${fdt_addr}  gsap-v2/armada-7040-db-GSAP.dtb; booti ${loadaddr} - ${fdt_addr}
 
=======================================================================================================

** histroy

Marvell> setenv bootcmd_bak run get_images\; run set_bootargs\; booti \$kernel_addr \$ramfs_addr \$fdt_addr
Marvell> printevn bootcmd_bak
  bootcmd_bak=run get_images; run set_bootargs; booti $kernel_addr $ramfs_addr $fdt_addr
Marvell> setenv bootcmd run sataboot
Mravell> saveenv

=======================================================================================================

** Error in reset

Marvell>> reset
resetting ...
PANIC in EL3 at x30 = 0x0000000004023238
x0 =            0x00000000f06f0084
x1 =            0x0000000004023220
x2 =            0x000000007ffa0ef0
x3 =            0x0000000004027050
x4 =            0x000000007ffa0e90
x5 =            0x0000000004027004
x6 =            0x000000007ffa0ec0
x7 =            0x000000007f6e9d21
x8 =            0x0000000000000000
x9 =            0x0000000000000008
x10 =           0x0000000000000000
x11 =           0x000000000402bea0
x12 =           0x000000000402d640
x13 =           0x0000000000000001
x14 =           0x000000000402ed48
x15 =           0x0000000004025930
x16 =           0x00000000600003c9
x17 =           0x000000007ffa34a4
x18 =           0x0000000000000731
x19 =           0x000000000402f000
x20 =           0x0000000000000000
x21 =           0x0000000000000000
x22 =           0x000000007f6e9d20
x23 =           0x000000007ffb1640
x24 =           0x0000000000000001
x25 =           0x0000000000000000
x26 =           0x0000000000000000
x27 =           0x0000000000000000
x28 =           0x000000007f6ec480
x29 =           0x000000000402d5e0
scr_el3 =               0x0000000000000731
sctlr_el3 =             0x0000000000cd183f
cptr_el3 =              0x0000000000000000
tcr_el3 =               0x0000000080803520
daif =          0x00000000000002c0
mair_el3 =              0x00000000004404ff
spsr_el3 =              0x00000000600003c9
elr_el3 =               0x000000007ffa34a4
ttbr0_el3 =             0x000000000402ece0
esr_el3 =               0x000000005e000000
far_el3 =               0x0000000000000000
spsr_el1 =              0x0000000000000000
elr_el1 =               0x0000000000000000
spsr_abt =              0x0000000000000000
spsr_und =              0x0000000000000000
spsr_irq =              0x0000000000000000
spsr_fiq =              0x0000000000000000
sctlr_el1 =             0x0000000030d00800
actlr_el1 =             0x0000000000000000
cpacr_el1 =             0x0000000000000000
csselr_el1 =            0x0000000000000000
sp_el1 =                0x0000000000000000
esr_el1 =               0x0000000000000000
ttbr0_el1 =             0x0000000000000000
ttbr1_el1 =             0x0000000000000000
mair_el1 =              0x0000000000000000
amair_el1 =             0x0000000000000000
tcr_el1 =               0x0000000000000000
tpidr_el1 =             0x0000000000000000
tpidr_el0 =             0x0000000000000000
tpidrro_el0 =           0x0000000000000000
dacr32_el2 =            0x0000000000000000
ifsr32_el2 =            0x0000000000000000
par_el1 =               0x0000000000000000
mpidr_el1 =             0x0000000080000000
afsr0_el1 =             0x0000000000000000
afsr1_el1 =             0x0000000000000000
contextidr_el1 =                0x0000000000000000
vbar_el1 =              0x0000000000000000
cntp_ctl_el0 =          0x0000000000000000
cntp_cval_el0 =         0x0000000000000000
cntv_ctl_el0 =          0x0000000000000000
cntv_cval_el0 =         0x0000000000000000
cntkctl_el1 =           0x0000000000000000
fpexc32_el2 =           0x0000000000000700
sp_el0 =                0x000000000402d5e0
isr_el1 =               0x0000000000000000
cpuectlr_el1 =          0x0000001b00000040
cpumerrsr_el1 =         0x0000000000000000
l2merrsr_el1 =          0x0000000000000000

==========================================================================================
