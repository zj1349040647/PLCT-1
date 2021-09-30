# PLCT-1
观测点1的可见交付物
要在自己的机器上使用qemu构建一个虚拟机Fedora并在Fedora上跑riscv-gnu-toolchain。

1. 构建qemu环境
1.0. 错误示范安装qemu：
下载qemu：

sudo apt-get install qemu

下载下来的版本是2.11.1，其实这样未经配置目标的qemu列表中是没有我们需要的qemu-system-riscv64这个模拟器的，install之后可以在任意目录下 qemu-【TAB】【TAB】查看，所以，老老实实手动安装：

1.1. 正确示范：
新机先安装很多包：

$ sudo apt-get install gcc libc6-dev pkg-config bridge-utils uml-utilities zlib1g-dev libglib2.0-dev autoconf automake libtool libsdl1.2-dev
1
然后下载qemu，然后新建文件夹配置我们需要的模式，这里我们两种模式目标都配置上，即系统仿真模式用户仿真模式：

$ git clone https://git.qemu.org/git/qemu.git
$ cd qemu & mkdir build & cd build 
$ ../configure --target-list=riscv64-linux-user,riscv64-softmmu  [--prefix=INSTALL_LOCATION] 

安装路径因为我就是装在这个文件夹下面，所以就没配置，riscv-64-linux-user为用户模式，可以运行基于riscv指令集编译的程序文件,softmmu为镜像模拟器。
然后报错：

显然是缺某个包，apt-cache search pixman查询，得知缺libpixman-1-dev;装上，然后再configure，即配置成功，然后直接构建：

make 

此为构建成功，然后安装：

make install

安装成功，在构建目录下的bin下面就能找到我们需要的qemu-system-riscv64：

此时直接qemu-system-riscv64 --version 还是查不到的，需要：

export PATH=$PATH:/home/zhangjun/qemu/build/bin  

或者将之加进~/.bashrc文件中。此后可以qemu- [tab] [tab] 可以看到有了我们要的 qemu-system-riscv64，安装qemu正式完成；

2. 使用qemu模拟Fedora运行环境
有了qemu之后，下一步下载Fedora镜像准备在qemu上运行：
先安装libguestfs-tools-c（虚拟机磁盘管理工具），但是Ubuntu中这个工具叫libguestfs-tools,所以安装完成之后才能正常配置模拟虚拟机环境；
然后，参见：https://fedoraproject.org/wiki/Architectures/RISC-V/Installing

wget https://dl.fedoraproject.org/pub/alt/risc-v/repo/virt-builder-images/images/Fedora-Minimal-Rawhide-20200108.n.0-sda.raw.xz  
wget https://dl.fedoraproject.org/pub/alt/risc-v/repo/virt-builder-images/images/Fedora-Minimal-Rawhide-20200108.n.0-fw_payload-uboot-qemu-virt-smode.elf 

然后是解压下载下来的Fedora-Minimal-Rawhide-20200108.n.0-sda.raw.xz：

unxz Fedora-Minimal-Rawhide-20200108.n.0-sda.raw.xz

2.1 为虚拟机先扩容，再使用qemu模拟
解压之后，切记，不要直接调用模拟器直接配置虚拟机，那样打开虚拟机你会发现dev/sda4 空间只有3.9G!!!所以切记是先为raw图像文件扩容再启动，扩容的思想无非就是新建一个扩容图像文件取代原先的图像：

truncate -r Fedora-Minimal-Rawhide-20200108.n.0-sda.raw expanded.raw
truncate -s 40G expanded.raw
virt-resize -v -x --expand /dev/vda4 Fedora-Minimal-Rawhide-20200108.n.0-sda.raw expanded.raw
virt-filesystems --long -h --all -a expanded.raw
virt-df -h -a expanded.raw

期间可能会遇到缺少某些工具，根据提示安装即可。过程中可能会遇到失败情况，一般加个sudo或者根据报错提示配置个环境变量即可：
 
最终扩容成功启动Fedora的命令：

qemu-system-riscv64    \
 -nographic   \
 -machine virt   \
 -smp 8     \
 -m 8G     \
 -kernel Fedora-Minimal-Rawhide-20200108.n.0-fw_payload-uboot-qemu-virt-smode.elf     \
 -bios none     \
 -object rng-random,filename=/dev/urandom,id=rng0     \
 -device virtio-rng-device,rng=rng0     \
 -device virtio-blk-device,drive=hd0     \
 -drive file=expanded.raw,format=raw,id=hd0     \
 -device virtio-net-device,netdev=usernet  \
 -fsdev local,security_model=passthrough,id=fsdev-fs0,path=/tmp   \
 -netdev user,id=usernet,hostfwd=tcp::10000-:22

加载页面如期而至：

登录界面：

这里简单注意根据上面的提示输入用户名和密码。。。
正确配置之后查看内存情况：


2.2 共享文件目录以略过扩容
另外，也可以使用共享目录的思想，在qemu构建时需要加入–enable-virtfs选项，于是重新构建qemu，尝试打开共享文件目录功能与将要模拟的Fedora虚拟机共享文件，以解决虚拟机的磁盘空间问题。
在第一节中configure的选项中加入–enable-virtfs：

../configure --target-list=riscv64-linux-user,riscv64-softmmu --enable-virtfs  --enable-kvm #（后者也顺便加上）

重新安装qemu之后，在启动qemu模拟Fedora的初始化配置中加入：-fsdev local,security_model=passthrough,id=fsdev-fs0,path=/tmp -device virtio-9p-pci,id=fs0,fsdev=fsdev-fs0,mount_tag=test：
如下：

qemu-system-riscv64     \
-nographic     \
-machine virt     \
-smp 8     \
-m 8G     \
-kernel Fedora-Minimal-Rawhide-20200108.n.0-fw_payload-uboot-qemu-virt-smode.elf     \
-bios none     \
-object rng-random,filename=/dev/urandom,id=rng0     \
-device virtio-rng-device,rng=rng0     \
-device virtio-blk-device,drive=hd0     \
-drive file=expanded.raw,format=raw,id=hd0     \
-device virtio-net-device,netdev=usernet  \
-fsdev local,security_model=passthrough,id=fsdev0,path=**/tmp/share**  \
-device virtio-9p-pci,fsdev=fsdev0,mount_tag=hostshare
-netdev user,id=usernet,hostfwd=tcp::10000-:22

加粗部分换成自己要共享给Fedora的主机的目录。
进去之后：

mount -t 9p -o trans=virtio,version=9p2000.L hostshare /mnt/ #（报错没有权限的话加个sudo咯）
 
mount操作报错，先是需要sudo，其次是报错：
qemu-system-riscv64: warning: 9p: degraded performance: a reasonable high msize should be chosen on client/guest side (chosen msize is <= 8192). See https://wiki.qemu.org/Documentation/9psetup#msize for details.
警告而已，不管它，直接查看mnt:

此时已经完成共享文件夹，则无需再在Fedora上clonetoolchain。
注：关于以上命令的解释，可以看：关于命令的解释，可以看：
http://blog.leanote.com/post/7wlnk13/%E5%88%9B%E5%BB%BAKVM%E8%99%9A%E6%8B%9F%E6%9C%BA

下一次要进Fedora自然要重新启动qemu连接，重新登录虚拟机。
有时重新启动qemu会报错，则改变端口重新登录即可，若此时确实需要重新配置新的虚拟机，则可以删除掉两个文件重新下载或连带协议号一起改变。


3. 在Fedora上跑riscv-gnu-toolchain
这里跑toolchain教程很多，可以看我的上一篇：
https://blog.csdn.net/Z_july/article/details/120408700
注：新机一定要先安装各种包，不然每次跑半个小时几个小时报个错的，很烦的.
原文链接：https://blog.csdn.net/Z_july/article/details/120499121
