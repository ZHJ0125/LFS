# 安装LFS过程记录

## 1. 查看您的主机系统是否具有所有适当的版本
```sh
cat > version-check.sh << "EOF"
#!/bin/bash
# Simple script to list version numbers of critical development tools
export LC_ALL=C
bash --version | head -n1 | cut -d" " -f2-4
MYSH=$(readlink -f /bin/sh)
echo "/bin/sh -> $MYSH"
echo $MYSH | grep -q bash || echo "ERROR: /bin/sh does not point to bash"
unset MYSH

echo -n "Binutils: "; ld --version | head -n1 | cut -d" " -f3-
bison --version | head -n1

if [ -h /usr/bin/yacc ]; then
  echo "/usr/bin/yacc -> `readlink -f /usr/bin/yacc`";
elif [ -x /usr/bin/yacc ]; then
  echo yacc is `/usr/bin/yacc --version | head -n1`
else
  echo "yacc not found" 
fi

bzip2 --version 2>&1 < /dev/null | head -n1 | cut -d" " -f1,6-
echo -n "Coreutils: "; chown --version | head -n1 | cut -d")" -f2
diff --version | head -n1
find --version | head -n1
gawk --version | head -n1

if [ -h /usr/bin/awk ]; then
  echo "/usr/bin/awk -> `readlink -f /usr/bin/awk`";
elif [ -x /usr/bin/awk ]; then
  echo awk is `/usr/bin/awk --version | head -n1`
else 
  echo "awk not found" 
fi

gcc --version | head -n1
g++ --version | head -n1
ldd --version | head -n1 | cut -d" " -f2-  # glibc version
grep --version | head -n1
gzip --version | head -n1
cat /proc/version
m4 --version | head -n1
make --version | head -n1
patch --version | head -n1
echo Perl `perl -V:version`
python3 --version
sed --version | head -n1
tar --version | head -n1
makeinfo --version | head -n1  # texinfo version
xz --version | head -n1

echo 'int main(){}' > dummy.c && g++ -o dummy dummy.c
if [ -x dummy ]
  then echo "g++ compilation OK";
  else echo "g++ compilation failed"; fi
rm -f dummy.c dummy
EOF

bash version-check.sh
```

安装缺失的软件包，直到脚本检查无误

```sh
# 安装缺失的软件包
zhj@ubuntu:~$ sudo apt-get install bison gawk gcc g++ m4 make texinfo
# 软连接重新定向
zhj@ubuntu:~$ sudo rm /bin/sh && sudo ln -s /bin/bash /bin/sh
```

## 2. 挂载硬盘

构建 LFS 系统比较推荐的方法是使用可用的空分区，我这里使用VMware的虚拟硬盘，为LFS构建系统分配了一块20G大小的硬盘空间。

```sh
zhj@ubuntu:~$ ls -l /dev/sd*
brw-rw---- 1 root disk 8,  0 Dec 17 21:54 /dev/sda
brw-rw---- 1 root disk 8,  1 Dec 17 21:54 /dev/sda1
brw-rw---- 1 root disk 8, 16 Dec 17 21:54 /dev/sdb
# sda是我的Ubuntu系统安装盘，sdb是准备编译LFS的硬盘
```

为硬盘sdb重新分区

```sh
zhj@ubuntu:~$ sudo fdisk /dev/sdb
[sudo] password for zhj: 

Welcome to fdisk (util-linux 2.31.1).
Changes will remain in memory only, until you decide to write them.
Be careful before using the write command.

Device does not contain a recognized partition table.
Created a new DOS disklabel with disk identifier 0x418f8c1c.

Command (m for help): d
No partition is defined yet!
Could not delete partition 94115211317017

Command (m for help): n
Partition type
   p   primary (0 primary, 0 extended, 4 free)
   e   extended (container for logical partitions)
Select (default p): p
Partition number (1-4, default 1): 
First sector (2048-41943039, default 2048): 
Last sector, +sectors or +size{K,M,G,T,P} (2048-41943039, default 41943039): 

Created a new partition 1 of type 'Linux' and of size 20 GiB.

Command (m for help): w
The partition table has been altered.
Calling ioctl() to re-read partition table.
Syncing disks.

zhj@ubuntu:~$ 
```

格式化sdb硬盘

```sh
zhj@ubuntu:~$ partprobe
zhj@ubuntu:~$ sudo mkfs -v -t ext4 /dev/sdb1
mke2fs 1.44.1 (24-Mar-2018)
fs_types for mke2fs.conf resolution: 'ext4'
Filesystem label=
OS type: Linux
Block size=4096 (log=2)
Fragment size=4096 (log=2)
Stride=0 blocks, Stripe width=0 blocks
1310720 inodes, 5242624 blocks
262131 blocks (5.00%) reserved for the super user
First data block=0
Maximum filesystem blocks=2153775104
160 block groups
32768 blocks per group, 32768 fragments per group
8192 inodes per group
Filesystem UUID: 48c83dcb-ed96-4f29-ae2b-7c1eb8e39c45
Superblock backups stored on blocks: 
	32768, 98304, 163840, 229376, 294912, 819200, 884736, 1605632, 2654208, 
	4096000

Allocating group tables: done                            
Writing inode tables: done                            
Creating journal (32768 blocks): done
Writing superblocks and filesystem accounting information: done   

zhj@ubuntu:~$
```

设置硬盘挂载点

```sh
# 将sdb硬盘挂载到/mnt/lfs目录，接下来会在/mnt/lfs目录进行Linux系统的编译
zhj@ubuntu:~$ export LFS=/mnt/lfs
zhj@ubuntu:~$ sudo mkdir -pv $LFS
mkdir: created directory '/mnt/lfs'
zhj@ubuntu:~$ sudo mount -v -t ext4 /dev/sdb1 $LFS
mount: /dev/sdb1 mounted on /mnt/lfs.
```

## 3. 下载软件包和补丁
```sh
# 为软件创建目录
zhj@ubuntu:~$ sudo mkdir -v $LFS/sources
mkdir: created directory '/mnt/lfs/sources'
zhj@ubuntu:~$ sudo chmod -v a+wt $LFS/sources
mode of '/mnt/lfs/sources' changed from 0755 (rwxr-xr-x) to 1777 (rwxrwxrwt)
# 下载所有包和补丁
zhj@ubuntu:~$ wget -O wget-list https://linux.cn/lfs/LFS-BOOK-7.7-systemd/wget-list-LFS7.7-systemd-USTC
zhj@ubuntu:~$ sudo wget --input-file=./wget-list --continue --directory-prefix=$LFS/sources
# 由于网络限制，建议使用中科大的镜像软件源
https://mirrors.ustc.edu.cn/lfs/lfs-packages/10.0/
# 下载过程还是很快的，完成后显示内容如下
FINISHED --2021-12-21 09:06:17--
Total wall clock time: 44s
Downloaded: 92 files, 423M in 41s (10.4 MB/s)
# 软件包下载完成后进行性完整性校验
zhj@ubuntu:~$ pushd $LFS/sources
/mnt/lfs/sources ~
zhj@ubuntu:/mnt/lfs/sources$   md5sum -c md5sums
acl-2.2.53.tar.gz: OK
attr-2.4.48.tar.gz: OK
autoconf-2.69.tar.xz: OK
automake-1.16.2.tar.xz: OK
bash-5.0.tar.gz: OK
bc-3.1.5.tar.xz: OK
binutils-2.35.tar.xz: OK
bison-3.7.1.tar.xz: OK
bzip2-1.0.8.tar.gz: OK
check-0.15.2.tar.gz: OK
coreutils-8.32.tar.xz: OK
dbus-1.12.20.tar.gz: OK
dejagnu-1.6.2.tar.gz: OK
diffutils-3.7.tar.xz: OK
e2fsprogs-1.45.6.tar.gz: OK
elfutils-0.180.tar.bz2: OK
expat-2.2.9.tar.xz: OK
expect5.45.4.tar.gz: OK
file-5.39.tar.gz: OK
findutils-4.7.0.tar.xz: OK
flex-2.6.4.tar.gz: OK
gawk-5.1.0.tar.xz: OK
gcc-10.2.0.tar.xz: OK
gdbm-1.18.1.tar.gz: OK
gettext-0.21.tar.xz: OK
glibc-2.32.tar.xz: OK
gmp-6.2.0.tar.xz: OK
gperf-3.1.tar.gz: OK
grep-3.4.tar.xz: OK
groff-1.22.4.tar.gz: OK
grub-2.04.tar.xz: OK
gzip-1.10.tar.xz: OK
iana-etc-20200821.tar.gz: OK
inetutils-1.9.4.tar.xz: OK
intltool-0.51.0.tar.gz: OK
iproute2-5.8.0.tar.xz: OK
kbd-2.3.0.tar.xz: OK
kmod-27.tar.xz: OK
less-551.tar.gz: OK
libcap-2.42.tar.xz: OK
libffi-3.3.tar.gz: OK
libpipeline-1.5.3.tar.gz: OK
libtool-2.4.6.tar.xz: OK
linux-5.8.3.tar.xz: OK
m4-1.4.18.tar.xz: OK
make-4.3.tar.gz: OK
man-db-2.9.3.tar.xz: OK
man-pages-5.08.tar.xz: OK
meson-0.55.0.tar.gz: OK
mpc-1.1.0.tar.gz: OK
mpfr-4.1.0.tar.xz: OK
ncurses-6.2.tar.gz: OK
ninja-1.10.0.tar.gz: OK
openssl-1.1.1g.tar.gz: OK
patch-2.7.6.tar.xz: OK
perl-5.32.0.tar.xz: OK
pkg-config-0.29.2.tar.gz: OK
procps-ng-3.3.16.tar.xz: OK
psmisc-23.3.tar.xz: OK
Python-3.8.5.tar.xz: OK
python-3.8.5-docs-html.tar.bz2: OK
readline-8.0.tar.gz: OK
sed-4.8.tar.xz: OK
shadow-4.8.1.tar.xz: OK
systemd-246.tar.gz: OK
systemd-man-pages-246.tar.xz: OK
tar-1.32.tar.xz: OK
tcl8.6.10-src.tar.gz: OK
tcl8.6.10-html.tar.gz: OK
texinfo-6.7.tar.xz: OK
tzdata2020a.tar.gz: OK
util-linux-2.36.tar.xz: OK
vim-8.2.1361.tar.gz: OK
XML-Parser-2.46.tar.gz: OK
xz-5.2.5.tar.xz: OK
zlib-1.2.11.tar.xz: OK
zstd-1.4.5.tar.gz: OK
bash-5.0-upstream_fixes-1.patch: OK
bzip2-1.0.8-install_docs-1.patch: OK
coreutils-8.32-i18n-1.patch: OK
glibc-2.32-fhs-1.patch: OK
kbd-2.3.0-backspace-1.patch: OK
zhj@ubuntu:/mnt/lfs/sources$ popd
~
zhj@ubuntu:~$ 
```

## 4. 创建编译目录
```sh
# 为root用户设置密码并切换到root用户
zhj@ubuntu:~$ sudo passwd 
Enter new UNIX password: 
Retype new UNIX password: 
passwd: password updated successfully
zhj@ubuntu:~$ su 
Password: 
root@ubuntu:/home/zhj# 

# 创建LFS文件系统
root@ubuntu:~# mkdir -pv $LFS/{bin,etc,lib,sbin,usr,var}
mkdir: created directory '/mnt/lfs/bin'
mkdir: created directory '/mnt/lfs/etc'
mkdir: created directory '/mnt/lfs/lib'
mkdir: created directory '/mnt/lfs/sbin'
mkdir: created directory '/mnt/lfs/usr'
mkdir: created directory '/mnt/lfs/var'
# 为x86_64架构的机器创建目录
root@ubuntu:~# case $(uname -m) in
>   x86_64) mkdir -pv $LFS/lib64 ;;
> esac
mkdir: created directory '/mnt/lfs/lib64'
root@ubuntu:~# 
# 软件包和补丁下载好之后，我们需要编译这些软件，编译过程中产生的文件放在 $LFS/tools 目录
root@ubuntu:~$ mkdir -pv $LFS/tools
```

## 5. 创建lfs新用户
```sh
root@ubuntu:~# groupadd lfs
root@ubuntu:~# useradd -s /bin/bash -g lfs -m -k /dev/null lfs
root@ubuntu:~# passwd lfs
Enter new UNIX password: 
Retype new UNIX password: 
passwd: password updated successfully
# 授予lfs完全访问所有目录的权限
root@ubuntu:~# chown -v lfs $LFS/{usr,lib,var,etc,bin,sbin,tools}
changed ownership of '/mnt/lfs/usr' from root to lfs
changed ownership of '/mnt/lfs/lib' from root to lfs
changed ownership of '/mnt/lfs/var' from root to lfs
changed ownership of '/mnt/lfs/etc' from root to lfs
changed ownership of '/mnt/lfs/bin' from root to lfs
changed ownership of '/mnt/lfs/sbin' from root to lfs
changed ownership of '/mnt/lfs/tools' from root to lfs
root@ubuntu:~# case $(uname -m) in
>   x86_64) chown -v lfs $LFS/lib64 ;;
> esac
changed ownership of '/mnt/lfs/lib64' from root to lfs
root@ubuntu:~# chown -v lfs $LFS/sources
changed ownership of '/mnt/lfs/sources' from root to lfs
# 切换到lfs用户
root@ubuntu:~# su - lfs
lfs@ubuntu:~$ 
```

## 5. 配置lfs用户的bash
```sh
lfs@ubuntu:~$ pwd
/home/lfs
# 设置LFS用户的环境变量
cat > /home/lfs/.bash_profile << "EOF"
exec env -i HOME=$HOME TERM=$TERM PS1='\u:\w\$ ' /bin/bash
EOF
# 设置bash配置文件
cat > /home/lfs/.bashrc << "EOF"
set +h
umask 022
LFS=/mnt/lfs
LC_ALL=POSIX
LFS_TGT=$(uname -m)-lfs-linux-gnu
PATH=/usr/bin
if [ ! -L /bin ]; then PATH=/bin:$PATH; fi
PATH=$LFS/tools/bin:$PATH
export LFS LC_ALL LFS_TGT PATH
EOF
# 要确保lfs用户的环境干净，请检查 /etc/bash.bashrc 是否存在，如果存在，请将其移除
lfs@ubuntu:~$ su root
Password: 
root@ubuntu:/home/lfs# [ ! -e /etc/bash.bashrc ] || mv -v /etc/bash.bashrc /etc/bash.bashrc.NOUSE
renamed '/etc/bash.bashrc' -> '/etc/bash.bashrc.NOUSE'
root@ubuntu:/home/lfs# 
# 使配置生效
root@ubuntu:/home/lfs# source /home/lfs/.bash_profile
root@ubuntu:/home/lfs# su - lfs
lfs:~$ pwd
/home/lfs
lfs:~$ 
# 如果终端的样式为 lfs:~$ 说明配置成功
```

## 6. 查看CPU个数
```sh
lfs:~$ cat /proc/cpuinfo| grep "physical id"| sort| uniq| wc -l
4
# 我的Ubuntu使用4核心CPU，因此编译时可以使用 make -j4
# 为方便，可以直接将配置写入环境变量
lfs:~$ export MAKEFLAGS='-j4'
```

## 7. 构建交叉编译工具链

> ！从这部分开始构建新系统的真正工作，它需要非常小心以确保遵循说明，就像书中展示的那样。

在安装并配置好宿主机之后，我们就可以开始构建临时系统了！

构建该最小系统有两个步骤：

* 第一步是构建一个宿主系统无关的新工具链（编译器、汇编器、链接器、库和一些有用的工具）
* 第二步则是使用该工具链构建其它的基础工具。
* 此处编译的软件包都是临时性的，编译得到的文件将被安装在目录 `$LFS/tools`，以使其与之后安装的文件和宿主系统生成的目录分开

‼️ 重要提示：

每一个包编译的过程如下：

> 1. 把所有软件包和补丁都放到一个可以被访问的目录下，如`/mnt/lfs/sources/`。
> 2. 将当前目录切换至源码目录，即`/mnt/lfs/sources/`。
> 3. 对于每一个包：
>> 1. 使用`tar`命令，解压这个包。同时要确保你是`lfs`用户
>> 2. 切换到软件包解压后创建的目录
>> 3. 按照下文的提示进行构建
>> 4. 构建完成后，切换回源码目录，即`/mnt/lfs/sources/`
>> 5. 除非另有说明，否则删除这个包被解压后的文件夹

### 7.1 安装 Binutils-2.35

Binutils 软件包包括了一个链接器、汇编器和其它处理目标文件的工具。

```sh
# 切换到lfs用户，解压Binutils包
root@ubuntu:~# su - lfs
lfs:~$ echo $LFS
/mnt/lfs
lfs:~$ cd $LFS/sources
lfs:/mnt/lfs/sources$ tar xf binutils-2.35.tar.xz
lfs:/mnt/lfs/sources$ cd binutils-2.35
lfs:/mnt/lfs/sources/binutils-2.35$
# Binutils 手册建议在一个专门的编译目录里面编译
lfs:/mnt/lfs/sources/binutils-2.35$ mkdir -v build
mkdir: created directory 'build'
lfs:/mnt/lfs/sources/binutils-2.35$ cd build/
lfs:/mnt/lfs/sources/binutils-2.35/build$ 
# 输入以下配置
../configure --prefix=$LFS/tools       \
             --with-sysroot=$LFS        \
             --target=$LFS_TGT          \
             --disable-nls              \
             --disable-werror
# 配置完成后会输出：
configure: creating ./config.status
config.status: creating Makefile
```

各项配置参数的含义如下：
| 参数 | 含义 |
|-----|------|
| --prefix=$LFS/tools | 将binutils安装到 `$LFS/tools` 目录 |
| --with-sysroot=$LFS | 对于交叉编译，这会告诉构建系统根据需要在 `$LFS` 中查找目标系统库 |
| --target=$LFS_TGT   | 由于 `LFS_TGT` 变量中的机器描述与 `config.guess` 脚本返回的值略有不同，这项配置将告诉配置脚本调整 binutil 的构建系统以构建交叉链接器 |
| --disable-nls       | 这会禁用i18n，因为临时工具不需要国际化 |
| --disable-werror    | 在主机编译器发出警告的情况下继续构建任务 |

```sh
# 开始编译并安装 (大约耗时3分钟)
lfs:/mnt/lfs/sources/binutils-2.35/build$ time { make -j4 && make install; }
# 安装完成后提示：
done
make[5]: Leaving directory '/mnt/lfs/sources/binutils-2.35/build/binutils'
make[4]: Leaving directory '/mnt/lfs/sources/binutils-2.35/build/binutils'
make[3]: Leaving directory '/mnt/lfs/sources/binutils-2.35/build/binutils'
make[2]: Leaving directory '/mnt/lfs/sources/binutils-2.35/build/binutils'
make[1]: Leaving directory '/mnt/lfs/sources/binutils-2.35/build'

real    2m5.121s
user    3m45.467s
sys     1m7.392s
# 清除安装包
lfs:/mnt/lfs/sources/binutils-2.35/build$ cd ../..
lfs:/mnt/lfs/sources$ rm -rf binutils-2.35
```

### 7.2 安装cross-gcc

GCC 软件包是 GNU 编译器集合的一部分，其中包括 C 和 C++ 的编译器。

GCC 需要 GMP、 MPFR 和 MPC 软件包，在一些Linux发行版中可能并不包括这些软件包，因此需要把 GMP、 MPFR 和 MPC的软件包提前解压重命名并放到 GCC 源码目录下，在编译 GCC 的时候会自动调用这些软件包。

```sh
# 首先确保是 lfs 用户
lfs:/mnt/lfs/sources$ su - lfs
Password: 
lfs:~$ 
# 解压GCC软件包并进入源码目录
lfs:~$ cd $LFS/sources
lfs:/mnt/lfs/sources$ tar xf gcc-10.2.0.tar.xz
lfs:/mnt/lfs/sources$ cd gcc-10.2.0
# 解压依赖包并重命名
lfs:/mnt/lfs/sources/gcc-10.2.0$ tar -xf ../mpfr-4.1.0.tar.xz
lfs:/mnt/lfs/sources/gcc-10.2.0$ mv -v mpfr-4.1.0 mpfr
renamed 'mpfr-4.1.0' -> 'mpfr'
lfs:/mnt/lfs/sources/gcc-10.2.0$ tar -xf ../gmp-6.2.0.tar.xz
lfs:/mnt/lfs/sources/gcc-10.2.0$ mv -v gmp-6.2.0 gmp
renamed 'gmp-6.2.0' -> 'gmp'
lfs:/mnt/lfs/sources/gcc-10.2.0$ tar -xf ../mpc-1.1.0.tar.gz
lfs:/mnt/lfs/sources/gcc-10.2.0$ mv -v mpc-1.1.0 mpc
renamed 'mpc-1.1.0' -> 'mpc'
lfs:/mnt/lfs/sources/gcc-10.2.0$ 
# 在 x86_64 主机上，为64位设置默认目录名称库到"lib"
case $(uname -m) in
  x86_64)
    sed -e '/m64=/s/lib64/lib/' \
        -i.orig gcc/config/i386/t-linux64
 ;;
esac
# GCC 手册推荐创建专用的构建目录
lfs:/mnt/lfs/sources/gcc-10.2.0$ mkdir -v build
mkdir: created directory 'build'
lfs:/mnt/lfs/sources/gcc-10.2.0$ cd build/
lfs:/mnt/lfs/sources/gcc-10.2.0/build$
# 配置编译选项
../configure                                       \
    --target=$LFS_TGT                              \
    --prefix=$LFS/tools                            \
    --with-glibc-version=2.11                      \
    --with-sysroot=$LFS                            \
    --with-newlib                                  \
    --without-headers                              \
    --enable-initfini-array                        \
    --disable-nls                                  \
    --disable-shared                               \
    --disable-multilib                             \
    --disable-decimal-float                        \
    --disable-threads                              \
    --disable-libatomic                            \
    --disable-libgomp                              \
    --disable-libquadmath                          \
    --disable-libssp                               \
    --disable-libvtv                               \
    --disable-libstdcxx                            \
    --enable-languages=c,c++
# 配置完成后输出：
configure: creating ./config.status
config.status: creating Makefile
```

各项配置参数的含义如下：
| 参数 | 含义 |
|-----|------|
| --with-glibc-version=2.11 | 指定主机所需要的最小 Glibc 版本 |
| --with-newlib             | 由于尚无可用的 C 库，这项配置确保在构建 libgcc 时定义了 prevent_libc 常量。这可以防止编译任何需要 libc 支持的代码。 |
| --without-headers         | 创建完整的交叉编译器时，GCC 需要与目标系统兼容的标准头文件。我们是构建临时系统，不需要这些头文件 |
| --enable-initfini-array   | 强制使用某些在构建交叉编译器时需要但无法被检测到的内部数据结构 |
| --disable-shared          | 由于共享库需要 Glibc，它还未安装，所以强制 GCC 静态链接其内部库 |
| --disable-multilib        | 在 x86_64 上，LFS 不支持多库配置，为兼容 x86_64 而禁用此项 |
| --disable-decimal-float, --disable-threads, --disable-libatomic, --disable-libgomp, --disable-libquadmath, --disable-libssp, --disable-libvtv, --disable-libstdcxx     | 禁用对十进制浮点扩展、线程、libatomic、libgomp、libquadmath、libssp、libvtv 和 C++ 标准库的支持。这些功能在构建交叉编译器时将无法编译，并且对于临时的交叉编译任务不是必需的 |
| --enable-languages=c,c++  | 此项确保仅构建 C 和 C++ 编译器。这些是现在唯一需要的语言 |

```sh
# 编译gcc (大约耗时16分钟)
lfs:/mnt/lfs/sources/gcc-10.2.0/build$ time { make && make install; }
# 完成后输出信息：
make[2]: Leaving directory '/mnt/lfs/sources/gcc-10.2.0/build/gcc'
make[1]: Leaving directory '/mnt/lfs/sources/gcc-10.2.0/build'

real    15m17.557s
user    42m11.249s
sys     6m0.201s
# 准备 limits.h 头文件
cd ..
cat gcc/limitx.h gcc/glimits.h gcc/limity.h > \
  `dirname $($LFS_TGT-gcc -print-libgcc-file-name)`/install-tools/include/limits.h
# 删除编译的软件包
lfs:/mnt/lfs/sources/gcc-10.2.0$ cd ..
lfs:/mnt/lfs/sources$ rm -rf gcc-10.2.0
```

### 7.3 安装 Linux-5.8.3 API Headers

Linux 内核需要导出一个应用程序编程接口 (API) 供系统的 C 运行库 (例如 LFS 中的 Glibc) 使用。

```sh
# 切换到lfs用户并解压Linux压缩包
lfs:/mnt/lfs/sources$ echo $LFS
/mnt/lfs
lfs:/mnt/lfs/sources$ cd $LFS/sources
lfs:/mnt/lfs/sources$ tar xf linux-5.8.3.tar.xz 
lfs:/mnt/lfs/sources$ cd linux-5.8.3
lfs:/mnt/lfs/sources/linux-5.8.3$ 
# 确保软件包中没有过时的文件
lfs:/mnt/lfs/sources/linux-5.8.3$ make mrproper
# 从源代码中提取用户可见的头文件
lfs:/mnt/lfs/sources/linux-5.8.3$ make headers
lfs:/mnt/lfs/sources/linux-5.8.3$ find usr/include -name '.*' -delete
lfs:/mnt/lfs/sources/linux-5.8.3$ rm usr/include/Makefile
lfs:/mnt/lfs/sources/linux-5.8.3$ cp -rv usr/include $LFS/usr
# 安装完成后删除软件包
lfs:/mnt/lfs/sources/linux-5.8.3$ cd ..
lfs:/mnt/lfs/sources$ rm -rf linux-5.8.3
lfs:/mnt/lfs/sources$ 
```

### 7.4 安装 Glibc-2.32

Glibc 包含了主要的 C 库。该库提供了用于分配内存、搜索目录、打开和关闭文件、读取和写入文件、字符串处理、模式匹配、算术等的基本例程。

```sh
# 确保是 lfs 用户并解压软件包
lfs:/mnt/lfs/sources$ echo $LFS
/mnt/lfs
lfs:/mnt/lfs/sources$ cd $LFS/sources
lfs:/mnt/lfs/sources$ tar xf glibc-2.32.tar.xz 
lfs:/mnt/lfs/sources$ cd glibc-2.32
lfs:/mnt/lfs/sources/glibc-2.32$ 
# 对于 x86_64，创建一个动态链接器正常工作所必须的符号链接
case $(uname -m) in
    i?86)   ln -sfv ld-linux.so.2 $LFS/lib/ld-lsb.so.3
    ;;
    x86_64) ln -sfv ../lib/ld-linux-x86-64.so.2 $LFS/lib64
            ln -sfv ../lib/ld-linux-x86-64.so.2 $LFS/lib64/ld-lsb-x86-64.so.3
    ;;
esac
# 输出信息：
'/mnt/lfs/lib64/ld-linux-x86-64.so.2' -> '../lib/ld-linux-x86-64.so.2'
'/mnt/lfs/lib64/ld-lsb-x86-64.so.3' -> '../lib/ld-linux-x86-64.so.2'
# 安装兼容性补丁
lfs:/mnt/lfs/sources/glibc-2.32$ patch -Np1 -i ../glibc-2.32-fhs-1.patch
patching file Makeconfig
Hunk #1 succeeded at 245 (offset -5 lines).
patching file nscd/nscd.h
Hunk #1 succeeded at 161 (offset 49 lines).
patching file nss/db-Makefile
patching file sysdeps/generic/paths.h
patching file sysdeps/unix/sysv/linux/paths.h
lfs:/mnt/lfs/sources/glibc-2.32$ 
# 为Glibc创建编译文件夹
lfs:/mnt/lfs/sources/glibc-2.32$ mkdir -v build
mkdir: created directory 'build'
lfs:/mnt/lfs/sources/glibc-2.32$ cd build/
# 准备配置文件
../configure                             \
      --prefix=/usr                      \
      --host=$LFS_TGT                    \
      --build=$(../scripts/config.guess) \
      --enable-kernel=3.2                \
      --with-headers=$LFS/usr/include    \
      libc_cv_slibdir=/lib
# 配置完成后提示：
configure: creating ./config.status
config.status: creating config.make
config.status: creating Makefile
config.status: creating config.h
config.status: executing default commands
```
各项配置参数的含义如下：
| 参数 | 含义 |
|-----|------|
| --host=$LFS_TGT, --build=$(../scripts/config.guess) | 指定 Glibc 的构建系统将自身配置为交叉编译，使用 `$LFS/tools` 中的交叉链接器和交叉编译器。 |
| --enable-kernel=3.2       | 指定编译器支持的最低 Linux 内核版本 |
| --with-headers=$LFS/usr/include    | 指定 Glibc 根据 $LFS/usr/include 目录的头文件编译自己，以便它确切地知道内核具有哪些功能并可以相应地优化自己 |
| libc_cv_slibdir=/lib   | 确保库安装在 /lib 中，而不是 64 位机器上的默认 /lib64 中 |

```sh
# 开始编译 (大约耗时12分钟)，有报告说这个包在多核编译时可能会失败，可使用 make -j1 解决。
lfs:/mnt/lfs/sources/glibc-2.32$ time { make && make DESTDIR=$LFS install; }
# 输出信息：
make[1]: Leaving directory '/mnt/lfs/sources/glibc-2.32'

real    10m56.592s
user    20m29.037s
sys     5m59.230s
```

注意：接下来需要确保新工具链的基本功能（编译和链接）按预期工作，输入以下指令：

```sh
echo 'int main(){}' > dummy.c
$LFS_TGT-gcc dummy.c
readelf -l a.out | grep '/ld-linux'
```

如果输出以下信息，说明配置成功。如果输出的信息与下面不同，或者不输出信息，都表示配置失败。在进行下一步前，必须先解决该问题，务必保证配置成功。

```sh
lfs:/mnt/lfs/sources/glibc-2.32/build$ echo 'int main(){}' > dummy.c
lfs:/mnt/lfs/sources/glibc-2.32/build$ $LFS_TGT-gcc dummy.c
lfs:/mnt/lfs/sources/glibc-2.32/build$ readelf -l a.out | grep '/ld-linux'
      [Requesting program interpreter: /lib64/ld-linux-x86-64.so.2]
lfs:/mnt/lfs/sources/glibc-2.32/build$ 
```

在32位机器上，输出信息应该是：`[Requesting program interpreter: /lib/ld-linux.so.2]`。若配置无误，使用以下命令删除测试文件：

```sh
lfs:/mnt/lfs/sources/glibc-2.32/build$ rm -v dummy.c a.out
removed 'dummy.c'
removed 'a.out'
```

现在交叉编译工具链已经安装成功，接下来运行以下代码以完成 `limits.h` 头文件的安装。

```sh
lfs:/mnt/lfs/sources/glibc-2.32/build$ $LFS/tools/libexec/gcc/$LFS_TGT/10.2.0/install-tools/mkheaders
```

最后，删除软件包

```sh
lfs:/mnt/lfs/sources/glibc-2.32/build$ cd ../..
lfs:/mnt/lfs/sources$ rm -rf glibc-2.32
```

### 7.5 安装 Libstdc++

Libstdc++ 是 C++ 标准库。我们需要它才能编译 C++ 代码 (GCC 的一部分用 C++ 编写)。但在构建第一遍的 GCC时我们不得不暂缓安装它，因为它依赖于当时还没有安装到目标目录的 Glibc。

Libstdc++ 是 GCC 源码的一部分，因此需要先解压 GCC

```sh
lfs:/mnt/lfs/sources$ tar xf gcc-10.2.0.tar.xz
lfs:/mnt/lfs/sources$ cd gcc-10.2.0
```

为 Libstdc++ 创建专用的编译目录

```sh
lfs:/mnt/lfs/sources/gcc-10.2.0$ mkdir -v build
mkdir: created directory 'build'
lfs:/mnt/lfs/sources/gcc-10.2.0$ cd build
```

准备编译配置

```sh
../libstdc++-v3/configure           \
    --host=$LFS_TGT                 \
    --build=$(../config.guess)      \
    --prefix=/usr                   \
    --disable-multilib              \
    --disable-nls                   \
    --disable-libstdcxx-pch         \
    --with-gxx-include-dir=/tools/$LFS_TGT/include/c++/10.2.0

# 配置完成后输出以下信息：
64-lfs-linux-gnu/bits/gthr-posix.h
sed -e 's/\(UNUSED\)/_GLIBCXX_\1/g' \
    -e 's/\(GCC[ABCDEFGHIJKLMNOPQRSTUVWXYZ_]*_H\)/_GLIBCXX_\1/g' \
    -e 's/SUPPORTS_WEAK/__GXX_WEAK__/g' \
    -e 's/\([ABCDEFGHIJKLMNOPQRSTUVWXYZ_]*USE_WEAK\)/_GLIBCXX_\1/g' \
    -e 's,^#include "\(.*\)",#include <bits/\1>,g' \
    < /mnt/lfs/sources/gcc-10.2.0/libstdc++-v3/../libgcc/gthr-single.h > x86_64-lfs-linux-gnu/bits/gthr-default.h
lfs:/mnt/lfs/sources/gcc-10.2.0/build$ 
```

各项配置参数的含义如下：

| 参数 | 含义 |
|-----|------|
| --host=... | 指定使用我们刚刚构建的交叉编译器而不是 `/usr/bin` 中的交叉编译器 |
| --disable-libstdcxx-pch | 禁止安装现阶段不需要的预编译文件 |
| --with-gxx-include-dir=/tools/$LFS_TGT/include/c++/10.2.0 | 指定 C++ 编译器搜索标准包含文件的位置 |

开始编译并安装 Libstdc++ (大约耗时1分钟)

```sh
lfs:/mnt/lfs/sources/gcc-10.2.0/build$ time { make && make DESTDIR=$LFS install; }
```

安装完成后输出以下信息：

```sh
make[2]: Nothing to be done for 'install-data-am'.
make[2]: Leaving directory '/mnt/lfs/sources/gcc-10.2.0/build'
make[1]: Leaving directory '/mnt/lfs/sources/gcc-10.2.0/build'

real    0m50.626s
user    1m42.727s
sys     0m17.885s
lfs:/mnt/lfs/sources/gcc-10.2.0/build$ 
```

清理安装包

```sh
lfs:/mnt/lfs/sources/gcc-10.2.0/build$ cd ../..
lfs:/mnt/lfs/sources$ rm -rf gcc-10.2.0
```

## 8. 交叉编译临时工具

本章节展示了如何使用刚刚构建的交叉工具链去交叉编译基本实用程序。这些实用程序会被安装到其最终位置，但还不能使用，基本任务仍然依赖于宿主的工具。但是链接时会使用安装的库文件。

在下一章中，将进入chroot环境，到那时会使用这些实用程序。现在我们需要先构建本章中所涉及的所有包，因此还不能独立于主机系统。

再次重申， LFS 设置不当或是以 root 身份构建，可能会导致您的计算机无法使用。整个章节必须以用户 lfs 的身份完成。

### 8.1 安装 M4-1.4.18

M4 软件包内置了一个宏处理器

确保在 lfs 用户，解压并进入 M4 软件包

```sh
lfs:/mnt/lfs/sources$ su - lfs
Password: 
lfs:~$ cd $LFS/sources
lfs:/mnt/lfs/sources$ tar xf m4-1.4.18.tar.xz
lfs:/mnt/lfs/sources$ cd m4-1.4.18
```

做一些 glibc-2.28 所引入的修复

```sh
lfs:/mnt/lfs/sources/m4-1.4.18$ sed -i 's/IO_ftrylockfile/IO_EOF_SEEN/' lib/*.c
lfs:/mnt/lfs/sources/m4-1.4.18$ echo "#define _IO_IN_BACKUP 0x100" >> lib/stdio-impl.h
```

准备编译的配置文件

```sh
./configure --prefix=/usr   \
            --host=$LFS_TGT \
            --build=$(build-aux/config.guess)
# 配置完成后输出以下信息：
configure: creating ./config.status
config.status: creating Makefile
config.status: creating doc/Makefile
config.status: creating lib/Makefile
config.status: creating src/Makefile
config.status: creating tests/Makefile
config.status: creating checks/Makefile
config.status: creating examples/Makefile
config.status: creating lib/config.h
config.status: executing depfiles commands
config.status: executing stamp-h commands
lfs:/mnt/lfs/sources/m4-1.4.18$
```

开始编译并安装 (大约耗时15秒)

```sh
lfs:/mnt/lfs/sources/m4-1.4.18$ time { make && make DESTDIR=$LFS install; }
```

安装完成后，输出以下信息：
```sh
make[4]: Leaving directory '/mnt/lfs/sources/m4-1.4.18/tests'
make[3]: Leaving directory '/mnt/lfs/sources/m4-1.4.18/tests'
make[2]: Leaving directory '/mnt/lfs/sources/m4-1.4.18/tests'
make[1]: Leaving directory '/mnt/lfs/sources/m4-1.4.18'

real    0m11.779s
user    0m19.825s
sys     0m3.975s
```

清理软件包

```sh
lfs:/mnt/lfs/sources/m4-1.4.18$ cd ..
lfs:/mnt/lfs/sources$ rm -rf m4-1.4.18
```

### 8.2 安装 Ncurses-6.2

Ncurses 包含了独立于终端特性的字符屏幕处理函数库

解压软件包

```sh
lfs:/mnt/lfs/sources$ echo $LFS
/mnt/lfs
lfs:/mnt/lfs/sources$ cd $LFS/sources
lfs:/mnt/lfs/sources$ tar xf ncurses-6.2.tar.gz 
lfs:/mnt/lfs/sources$ cd ncurses-6.2
```

确保在配置过程中优先查找到 `gawk`

```sh
lfs:/mnt/lfs/sources/ncurses-6.2$ sed -i s/mawk// configure
```

运行以下命令，在宿主系统构建 `tic` 程序

```sh
mkdir build
pushd build
  ../configure
  make -C include
  make -C progs tic
popd

# 完成后输出提示信息：
make[1]: Leaving directory '/mnt/lfs/sources/ncurses-6.2/build/ncurses'
gcc ../objects/tic.o ../objects/dump_entry.o ../objects/tparm_type.o ../objects/transform.o -L../lib  -DHAVE_CONFIG_H -I../progs -I. -I../../progs -I../include -I../../progs/../include -D_DEFAULT_SOURCE -D_XOPEN_SOURCE=600 -DNDEBUG -O2 --param max-inline-insns-single=1200 -L../lib  -lncurses -lncurses    -o tic
make: Leaving directory '/mnt/lfs/sources/ncurses-6.2/build/progs'
```

准备配置编译信息

```sh
./configure --prefix=/usr                \
            --host=$LFS_TGT              \
            --build=$(./config.guess)    \
            --mandir=/usr/share/man      \
            --with-manpage-format=normal \
            --with-shared                \
            --without-debug              \
            --without-ada                \
            --without-normal             \
            --enable-widec

# 配置完成后输出以下信息：
** Configuration summary for NCURSES 6.2 20200212:

       extended funcs: yes
       xterm terminfo: xterm-new

        bin directory: /usr/bin
        lib directory: /usr/lib
    include directory: /usr/include
        man directory: /usr/share/man
   terminfo directory: /usr/share/terminfo

lfs:/mnt/lfs/sources/ncurses-6.2$ 
```

各项配置参数的含义如下：

| 参数 | 含义 |
|-----|------|
| --with-manpage-format=normal | 该项可以防止 Ncurses 安装压缩的手册页，如果宿主机的Linux发行版本身具有压缩的手册页，则可能会发生这种情况。 |
| --without-ada | 这项确保 Ncurses 不会构建对 Ada 编译器的支持，该编译器可能存在于宿主机上，但一旦我们进入 chroot 环境就将不可用。 |
| --enable-widec | 指定构建宽字符库（例如 libncursesw.so.6.2）而不是普通库（例如 libncurses.so.6.2）这些宽字符库可用于多字节和传统的 8 位语言环境，而普通库只能在 8 位语言环境中正常工作。宽字符库和普通库是源代码兼容的，但不是二进制兼容的。 |
| --without-normal | 禁止构建和安装大多数静态库 |

编译程序 (大约耗时20秒)

```sh
lfs:/mnt/lfs/sources/ncurses-6.2$ make

# 编译完成后输出信息：
make[1]: Leaving directory '/mnt/lfs/sources/ncurses-6.2/test'
cd misc && make DESTDIR="" RPATH_LIST="/usr/lib" all
make[1]: Entering directory '/mnt/lfs/sources/ncurses-6.2/misc'
WHICH_XTERM=xterm-new \
XTERM_KBS=BS \
datadir=/usr/share \
/bin/sh ./gen_edit.sh >run_tic.sed
echo '** adjusting tabset paths'
** adjusting tabset paths
sed -f run_tic.sed ../misc/terminfo.src >terminfo.tmp
make[1]: Leaving directory '/mnt/lfs/sources/ncurses-6.2/misc'
lfs:/mnt/lfs/sources/ncurses-6.2$ 
```

安装软件包 (大约耗时15秒)

```sh
lfs:/mnt/lfs/sources/ncurses-6.2$ make DESTDIR=$LFS TIC_PATH=$(pwd)/build/progs/tic install

# 安装后输出信息：
1750 entries written to /mnt/lfs/usr/share/terminfo
** built new /mnt/lfs/usr/share/terminfo
** sym-linked /mnt/lfs/usr/lib/terminfo for compatibility
installing std
installing stdcrt
installing vt100
installing vt300
finished install.data
/usr/bin/install -c ncurses-config /mnt/lfs/usr/bin/ncursesw6-config
make[1]: Leaving directory '/mnt/lfs/sources/ncurses-6.2/misc'
lfs:/mnt/lfs/sources/ncurses-6.2$ 
```

继续执行以下指令以完成安装

```sh
lfs:/mnt/lfs/sources/ncurses-6.2$ echo "INPUT(-lncursesw)" > $LFS/usr/lib/libncurses.so
```

将共享库移动到 `/lib` 目录，它们应该驻留在那里

```sh
lfs:/mnt/lfs/sources/ncurses-6.2$ mv -v $LFS/usr/lib/libncursesw.so.6* $LFS/lib
renamed '/mnt/lfs/usr/lib/libncursesw.so.6' -> '/mnt/lfs/lib/libncursesw.so.6'
renamed '/mnt/lfs/usr/lib/libncursesw.so.6.2' -> '/mnt/lfs/lib/libncursesw.so.6.2'
```

由于库被移动，会有一个符号链接指向不存在的文件。我们来重新创建它。

```sh
lfs:/mnt/lfs/sources/ncurses-6.2$ ln -sfv ../../lib/$(readlink $LFS/usr/lib/libncursesw.so) $LFS/usr/lib/libncursesw.so
'/mnt/lfs/usr/lib/libncursesw.so' -> '../../lib/libncursesw.so.6'
```

安装完成后清理软件包

```sh
lfs:/mnt/lfs/sources/ncurses-6.2$ cd ..
lfs:/mnt/lfs/sources$ rm -rf ncurses-6.2
```

### 8.3 安装 Bash-5.0

Bash 软件包包含了 Bourne-Again SHell

解压软件包

```sh
lfs:/mnt/lfs/sources$ echo $LFS
/mnt/lfs
lfs:/mnt/lfs/sources$ cd $LFS/sources
lfs:/mnt/lfs/sources$ tar xf bash-5.0.tar.gz
lfs:/mnt/lfs/sources$ cd bash-5.0
lfs:/mnt/lfs/sources/bash-5.0$ 
```

准备配置文件并编译安装 (大约耗时1分钟)

```sh
time { ./configure --prefix=/usr            \
            --build=$(support/config.guess) \
            --host=$LFS_TGT                 \
            --without-bash-malloc && make && make DESTDIR=$LFS install; }

# 安装完成后输出以下信息：
make[2]: Entering directory '/mnt/lfs/sources/bash-5.0'
mkdir -p -- /mnt/lfs/usr/include/bash
mkdir -p -- /mnt/lfs/usr/include/bash/builtins
mkdir -p -- /mnt/lfs/usr/include/bash/include
mkdir -p -- /mnt/lfs/usr/lib/pkgconfig
/usr/bin/install -c -m 644 ./support/bash.pc /mnt/lfs/usr/lib/pkgconfig/bash.pc
make[2]: Leaving directory '/mnt/lfs/sources/bash-5.0'
installing example loadable builtins in /mnt/lfs/usr/lib/bash
print
truefalse
sleep
finfo
logname
basename
dirname
fdflags
tty
pathchk
tee
head
mkdir
rmdir
printenv
id
whoami
uname
sync
push
ln
unlink
realpath
strftime
mypid
setpgid
seq
make[1]: Leaving directory '/mnt/lfs/sources/bash-5.0/examples/loadables'

real    1m11.409s
user    1m36.878s
sys     0m23.148s
```

各项配置参数的含义如下：

| 参数 | 含义 |
|-----|------|
| --without-bash-malloc | 此选项禁用 Bash 的内存分配 (malloc) 函数的使用，该函数已知会导致段错误。通过关闭此选项，Bash 将使用 Glibc 中更稳定的 malloc 函数。 |

将可执行文件移动到预期位置，并为使用 sh 作为 shell 的程序创建一个链接

```sh
lfs:/mnt/lfs/sources/bash-5.0$ mv $LFS/usr/bin/bash $LFS/bin/bash
lfs:/mnt/lfs/sources/bash-5.0$ ln -sv bash $LFS/bin/sh
'/mnt/lfs/bin/sh' -> 'bash'
```

安装完成后清理软件包

```sh
lfs:/mnt/lfs/sources/bash-5.0$ cd ..
lfs:/mnt/lfs/sources$ rm -rf bash-5.0
```

### 8.4 安装 Coreutils-8.32

Coreutils 包含了用于显示和设置基本系统特征的实用程序。

解压软件包

```sh
lfs:/mnt/lfs/sources$ echo $LFS
/mnt/lfs
lfs:/mnt/lfs/sources$ cd $LFS/sources
lfs:/mnt/lfs/sources$ tar xf coreutils-8.32.tar.xz
lfs:/mnt/lfs/sources$ cd coreutils-8.32
```

配置 Coreutils 并编译安装 (大约耗时2分钟)

```sh
time { ./configure --prefix=/usr              \
            --host=$LFS_TGT                   \
            --build=$(build-aux/config.guess) \
            --enable-install-program=hostname \
            --enable-no-install-program=kill,uptime && make && make DESTDIR=$LFS install; }

# 安装完成后输出以下信息：
make  install-exec-hook
make[4]: Entering directory '/mnt/lfs/sources/coreutils-8.32'
make[4]: Leaving directory '/mnt/lfs/sources/coreutils-8.32'
make[3]: Leaving directory '/mnt/lfs/sources/coreutils-8.32'
make[2]: Leaving directory '/mnt/lfs/sources/coreutils-8.32'
Making install in gnulib-tests
make[2]: Entering directory '/mnt/lfs/sources/coreutils-8.32/gnulib-tests'
make  install-recursive
make[3]: Entering directory '/mnt/lfs/sources/coreutils-8.32/gnulib-tests'
Making install in .
make[4]: Entering directory '/mnt/lfs/sources/coreutils-8.32/gnulib-tests'
make[5]: Entering directory '/mnt/lfs/sources/coreutils-8.32/gnulib-tests'
make[5]: Leaving directory '/mnt/lfs/sources/coreutils-8.32/gnulib-tests'
make[4]: Leaving directory '/mnt/lfs/sources/coreutils-8.32/gnulib-tests'
make[3]: Leaving directory '/mnt/lfs/sources/coreutils-8.32/gnulib-tests'
make[2]: Leaving directory '/mnt/lfs/sources/coreutils-8.32/gnulib-tests'
make[1]: Leaving directory '/mnt/lfs/sources/coreutils-8.32'

real    2m11.443s
user    2m40.951s
sys     0m51.252s
```

各项配置参数的含义如下：

| 参数 | 含义 |
|-----|------|
| --enable-install-program=hostname | 这使主机名二进制文件能够被构建和安装。默认情况下它是禁用的，但 Perl 测试套件需要它 |

接下来将程序移动到它们最终的预期位置。尽管在这个临时环境中这不是必需的，但我们必须这样做，因为某些程序对可执行位置进行了硬编码。

```sh
# 请逐条粘贴执行
mv -v $LFS/usr/bin/{cat,chgrp,chmod,chown,cp,date,dd,df,echo} $LFS/bin
mv -v $LFS/usr/bin/{false,ln,ls,mkdir,mknod,mv,pwd,rm} $LFS/bin
mv -v $LFS/usr/bin/{rmdir,stty,sync,true,uname} $LFS/bin
mv -v $LFS/usr/bin/{head,nice,sleep,touch} $LFS/bin
mv -v $LFS/usr/bin/chroot $LFS/usr/sbin
mkdir -pv $LFS/usr/share/man/man8
mv -v $LFS/usr/share/man/man1/chroot.1 $LFS/usr/share/man/man8/chroot.8
sed -i 's/"1"/"8"/' $LFS/usr/share/man/man8/chroot.8
```

安装完成后清理安装包

```sh
lfs:/mnt/lfs/sources/coreutils-8.32$ cd ..
lfs:/mnt/lfs/sources$ rm -rf coreutils-8.32
```

### 8.5 安装 Diffutils-3.7

Diffutils 包含了显示文件或目录之间差异的程序

解压软件包

```sh
lfs:/mnt/lfs/sources$ echo $LFS
/mnt/lfs
lfs:/mnt/lfs/sources$ cd $LFS/sources
lfs:/mnt/lfs/sources$ tar xf diffutils-3.7.tar.xz 
lfs:/mnt/lfs/sources$ cd diffutils-3.7
```

编译并安装 Diffutils (大约耗时1分钟)

```sh
time { ./configure --prefix=/usr --host=$LFS_TGT && make && make DESTDIR=$LFS install; }

# 安装完成后输出以下信息：
make[1]: Leaving directory '/mnt/lfs/sources/diffutils-3.7/po'
Making install in gnulib-tests
make[1]: Entering directory '/mnt/lfs/sources/diffutils-3.7/gnulib-tests'
make  install-recursive
make[2]: Entering directory '/mnt/lfs/sources/diffutils-3.7/gnulib-tests'
Making install in .
make[3]: Entering directory '/mnt/lfs/sources/diffutils-3.7/gnulib-tests'
make[4]: Entering directory '/mnt/lfs/sources/diffutils-3.7/gnulib-tests'
make[4]: Nothing to be done for 'install-exec-am'.
make[4]: Nothing to be done for 'install-data-am'.
make[4]: Leaving directory '/mnt/lfs/sources/diffutils-3.7/gnulib-tests'
make[3]: Leaving directory '/mnt/lfs/sources/diffutils-3.7/gnulib-tests'
make[2]: Leaving directory '/mnt/lfs/sources/diffutils-3.7/gnulib-tests'
make[1]: Leaving directory '/mnt/lfs/sources/diffutils-3.7/gnulib-tests'
make[1]: Entering directory '/mnt/lfs/sources/diffutils-3.7'
make[2]: Entering directory '/mnt/lfs/sources/diffutils-3.7'
make[2]: Nothing to be done for 'install-exec-am'.
make[2]: Nothing to be done for 'install-data-am'.
make[2]: Leaving directory '/mnt/lfs/sources/diffutils-3.7'
make[1]: Leaving directory '/mnt/lfs/sources/diffutils-3.7'

real    0m46.261s
user    0m40.908s
sys     0m14.536s
```

清理软件包

```sh
lfs:/mnt/lfs/sources/diffutils-3.7$ cd ..  
lfs:/mnt/lfs/sources$ rm -rf diffutils-3.7
```

### 8.6 安装 File-5.39

File 包含了用于确定指定文件类型的工具

解压软件包

```sh
lfs:/mnt/lfs/sources$ tar xf file-5.39.tar.gz 
lfs:/mnt/lfs/sources$ cd file-5.39
```

配置并编译安装 (大约耗时20秒)

```sh
time { ./configure --prefix=/usr --host=$LFS_TGT && make && make DESTDIR=$LFS install; }

# 安装完成后输出以下信息：
make[2]: Leaving directory '/mnt/lfs/sources/file-5.39/doc'
make[1]: Leaving directory '/mnt/lfs/sources/file-5.39/doc'
Making install in python
make[1]: Entering directory '/mnt/lfs/sources/file-5.39/python'
make[2]: Entering directory '/mnt/lfs/sources/file-5.39/python'
make[2]: Nothing to be done for 'install-exec-am'.
make[2]: Nothing to be done for 'install-data-am'.
make[2]: Leaving directory '/mnt/lfs/sources/file-5.39/python'
make[1]: Leaving directory '/mnt/lfs/sources/file-5.39/python'
make[1]: Entering directory '/mnt/lfs/sources/file-5.39'
make[2]: Entering directory '/mnt/lfs/sources/file-5.39'
make[2]: Nothing to be done for 'install-data-am'.
 /bin/mkdir -p '/mnt/lfs/usr/lib/pkgconfig'
 /usr/bin/install -c -m 644 libmagic.pc '/mnt/lfs/usr/lib/pkgconfig'
make[2]: Leaving directory '/mnt/lfs/sources/file-5.39'
make[1]: Leaving directory '/mnt/lfs/sources/file-5.39'

real    0m23.465s
user    0m23.628s
sys     0m7.370s
```

清理软件包

```sh
lfs:/mnt/lfs/sources/file-5.39$ cd ..
lfs:/mnt/lfs/sources$ rm -rf file-5.39
```

### 8.7 安装 Findutils-4.7.0

Findutils 包含了查找文件的程序，这些程序用于递归搜索目录树以及创建、维护和搜索数据库（通常比递归查找快，但如果数据库最近没有更新则不可靠）。

解压软件包

```sh
lfs:/mnt/lfs/sources$ tar xf findutils-4.7.0.tar.xz 
lfs:/mnt/lfs/sources$ cd findutils-4.7.0
```

配置并编译安装 (大约耗时1分钟)

```sh
time { ./configure --prefix=/usr   \
            --host=$LFS_TGT \
            --build=$(build-aux/config.guess) && make && make DESTDIR=$LFS install; }

# 安装完成后输出以下信息：
make[1]: Leaving directory '/mnt/lfs/sources/findutils-4.7.0/po'
Making install in m4
make[1]: Entering directory '/mnt/lfs/sources/findutils-4.7.0/m4'
make[2]: Entering directory '/mnt/lfs/sources/findutils-4.7.0/m4'
make[2]: Nothing to be done for 'install-exec-am'.
make[2]: Nothing to be done for 'install-data-am'.
make[2]: Leaving directory '/mnt/lfs/sources/findutils-4.7.0/m4'
make[1]: Leaving directory '/mnt/lfs/sources/findutils-4.7.0/m4'
Making install in gnulib-tests
make[1]: Entering directory '/mnt/lfs/sources/findutils-4.7.0/gnulib-tests'
make  install-recursive
make[2]: Entering directory '/mnt/lfs/sources/findutils-4.7.0/gnulib-tests'
Making install in .
make[3]: Entering directory '/mnt/lfs/sources/findutils-4.7.0/gnulib-tests'
make[4]: Entering directory '/mnt/lfs/sources/findutils-4.7.0/gnulib-tests'
make[4]: Nothing to be done for 'install-exec-am'.
make[4]: Nothing to be done for 'install-data-am'.
make[4]: Leaving directory '/mnt/lfs/sources/findutils-4.7.0/gnulib-tests'
make[3]: Leaving directory '/mnt/lfs/sources/findutils-4.7.0/gnulib-tests'
make[2]: Leaving directory '/mnt/lfs/sources/findutils-4.7.0/gnulib-tests'
make[1]: Leaving directory '/mnt/lfs/sources/findutils-4.7.0/gnulib-tests'
make[1]: Entering directory '/mnt/lfs/sources/findutils-4.7.0'
make[2]: Entering directory '/mnt/lfs/sources/findutils-4.7.0'
make[2]: Nothing to be done for 'install-exec-am'.
make[2]: Nothing to be done for 'install-data-am'.
make[2]: Leaving directory '/mnt/lfs/sources/findutils-4.7.0'
make[1]: Leaving directory '/mnt/lfs/sources/findutils-4.7.0'

real    0m58.110s
user    0m54.454s
sys     0m19.368s
```

将可执行文件移动到其最终预期位置

```sh
lfs:/mnt/lfs/sources/findutils-4.7.0$ mv -v $LFS/usr/bin/find $LFS/bin
renamed '/mnt/lfs/usr/bin/find' -> '/mnt/lfs/bin/find'
lfs:/mnt/lfs/sources/findutils-4.7.0$ sed -i 's|find:=${BINDIR}|find:=/bin|' $LFS/usr/bin/updatedb
```

清除软件包

```sh
lfs:/mnt/lfs/sources/findutils-4.7.0$ cd ..
lfs:/mnt/lfs/sources$ rm -rf findutils-4.7.0
```

### 8.8 安装 Gawk-5.1.0

Gawk 包含了操作文本文件的程序

解压软件包

```sh
lfs:/mnt/lfs/sources$ tar xf gawk-5.1.0.tar.xz 
lfs:/mnt/lfs/sources$ cd gawk-5.1.0
```

确保没有安装一些不需要的文件

```sh
lfs:/mnt/lfs/sources/gawk-5.1.0$ sed -i 's/extras//' Makefile.in
lfs:/mnt/lfs/sources/gawk-5.1.0$ 
```

配置并编译安装 (大约耗时1分钟)

```sh
time { ./configure --prefix=/usr   \
            --host=$LFS_TGT \
            --build=$(./config.guess) && make && make DESTDIR=$LFS install; }

# 安装完成后输出以下信息：
make[2]: Leaving directory '/mnt/lfs/sources/gawk-5.1.0/po'
Making install in test
make[2]: Entering directory '/mnt/lfs/sources/gawk-5.1.0/test'
make[3]: Entering directory '/mnt/lfs/sources/gawk-5.1.0/test'
make[3]: Nothing to be done for 'install-exec-am'.
make[3]: Nothing to be done for 'install-data-am'.
make[3]: Leaving directory '/mnt/lfs/sources/gawk-5.1.0/test'
make[2]: Leaving directory '/mnt/lfs/sources/gawk-5.1.0/test'
make[1]: Leaving directory '/mnt/lfs/sources/gawk-5.1.0'

real    0m49.920s
user    1m3.150s
sys     0m14.691s
```

清除软件包

```sh
lfs:/mnt/lfs/sources/gawk-5.1.0$ cd ..
lfs:/mnt/lfs/sources$ rm -rf gawk-5.1.0
```

### 8.9 安装 Grep-3.4

Grep 包含了在文件内容中进行搜索的程序

解压软件包

```sh
lfs:/mnt/lfs/sources$ tar xf grep-3.4.tar.xz 
lfs:/mnt/lfs/sources$ cd grep-3.4
```

配置并编译安装 (大约耗时1分钟)

```sh
time { ./configure --prefix=/usr   \
            --host=$LFS_TGT \
            --bindir=/bin && make && make DESTDIR=$LFS install; }

# 安装完成后输出以下信息：
make[2]: Leaving directory '/mnt/lfs/sources/grep-3.4/src'
make[1]: Leaving directory '/mnt/lfs/sources/grep-3.4/src'
Making install in tests
make[1]: Entering directory '/mnt/lfs/sources/grep-3.4/tests'
make[2]: Entering directory '/mnt/lfs/sources/grep-3.4/tests'
make[2]: Nothing to be done for 'install-exec-am'.
make[2]: Nothing to be done for 'install-data-am'.
make[2]: Leaving directory '/mnt/lfs/sources/grep-3.4/tests'
make[1]: Leaving directory '/mnt/lfs/sources/grep-3.4/tests'
Making install in gnulib-tests
make[1]: Entering directory '/mnt/lfs/sources/grep-3.4/gnulib-tests'
make  install-recursive
make[2]: Entering directory '/mnt/lfs/sources/grep-3.4/gnulib-tests'
Making install in .
make[3]: Entering directory '/mnt/lfs/sources/grep-3.4/gnulib-tests'
make[4]: Entering directory '/mnt/lfs/sources/grep-3.4/gnulib-tests'
make[4]: Nothing to be done for 'install-exec-am'.
make[4]: Nothing to be done for 'install-data-am'.
make[4]: Leaving directory '/mnt/lfs/sources/grep-3.4/gnulib-tests'
make[3]: Leaving directory '/mnt/lfs/sources/grep-3.4/gnulib-tests'
make[2]: Leaving directory '/mnt/lfs/sources/grep-3.4/gnulib-tests'
make[1]: Leaving directory '/mnt/lfs/sources/grep-3.4/gnulib-tests'
make[1]: Entering directory '/mnt/lfs/sources/grep-3.4'
make[2]: Entering directory '/mnt/lfs/sources/grep-3.4'
make[2]: Nothing to be done for 'install-exec-am'.
make[2]: Nothing to be done for 'install-data-am'.
make[2]: Leaving directory '/mnt/lfs/sources/grep-3.4'
make[1]: Leaving directory '/mnt/lfs/sources/grep-3.4'

real    0m44.616s
user    0m40.340s
sys     0m14.647s
```

清除软件包

```sh
lfs:/mnt/lfs/sources/grep-3.4$ cd ..
lfs:/mnt/lfs/sources$ rm -rf grep-3.4
```

### 8.10 安装 Gzip-1.10

Gzip 包含了压缩和解压缩文件的程序

解压软件包

```sh
lfs:/mnt/lfs/sources$ tar xf gzip-1.10.tar.xz 
lfs:/mnt/lfs/sources$ cd gzip-1.10
```

配置并编译安装 (大约耗时30秒)

```sh
time { ./configure --prefix=/usr --host=$LFS_TGT && make && make DESTDIR=$LFS install; }

# 编译安装完成后输出以下信息：
make[4]: Entering directory '/mnt/lfs/sources/gzip-1.10'
+ cd /mnt/lfs/usr/bin
+ rm -f uncompress
+ case remove-installed-links in
+ cd /mnt/lfs/usr/bin
+ rm -f uncompress
+ case install-exec-hook in
+ ln gunzip uncompress
make[4]: Leaving directory '/mnt/lfs/sources/gzip-1.10'
make[3]: Leaving directory '/mnt/lfs/sources/gzip-1.10'
make[2]: Leaving directory '/mnt/lfs/sources/gzip-1.10'
Making install in tests
make[2]: Entering directory '/mnt/lfs/sources/gzip-1.10/tests'
make[3]: Entering directory '/mnt/lfs/sources/gzip-1.10/tests'
make[3]: Nothing to be done for 'install-exec-am'.
make[3]: Nothing to be done for 'install-data-am'.
make[3]: Leaving directory '/mnt/lfs/sources/gzip-1.10/tests'
make[2]: Leaving directory '/mnt/lfs/sources/gzip-1.10/tests'
make[1]: Leaving directory '/mnt/lfs/sources/gzip-1.10'

real    0m26.124s
user    0m21.566s
sys     0m9.092s
```

将可执行文件移动到其最终预期位置

```sh
lfs:/mnt/lfs/sources/gzip-1.10$ mv -v $LFS/usr/bin/gzip $LFS/bin
renamed '/mnt/lfs/usr/bin/gzip' -> '/mnt/lfs/bin/gzip'
```

清除软件包

```sh
lfs:/mnt/lfs/sources/gzip-1.10$ cd ..
lfs:/mnt/lfs/sources$ rm -rf gzip-1.10
```

### 8.11 安装 Make-4.3

Make 用于控制从源文件生成包的可执行文件和其他非源文件

解压软件包

```sh
lfs:/mnt/lfs/sources$ tar xf make-4.3.tar.gz 
lfs:/mnt/lfs/sources$ cd make-4.3
```

配置并编译安装 (大约耗时20秒)

```sh
time { ./configure --prefix=/usr   \
            --without-guile        \
            --host=$LFS_TGT        \
            --build=$(build-aux/config.guess) && make && make DESTDIR=$LFS install; }

# 编译完成后输出以下信息：
make[1]: Leaving directory '/mnt/lfs/sources/make-4.3/po'
Making install in doc
make[1]: Entering directory '/mnt/lfs/sources/make-4.3/doc'
make[2]: Entering directory '/mnt/lfs/sources/make-4.3/doc'
make[2]: Nothing to be done for 'install-exec-am'.
 /bin/mkdir -p '/mnt/lfs/usr/share/info'
 /usr/bin/install -c -m 644 ./make.info ./make.info-1 ./make.info-2 '/mnt/lfs/usr/share/info'
 install-info --info-dir='/mnt/lfs/usr/share/info' '/mnt/lfs/usr/share/info/make.info'
make[2]: Leaving directory '/mnt/lfs/sources/make-4.3/doc'
make[1]: Leaving directory '/mnt/lfs/sources/make-4.3/doc'
make[1]: Entering directory '/mnt/lfs/sources/make-4.3'
make[2]: Entering directory '/mnt/lfs/sources/make-4.3'
 /bin/mkdir -p '/mnt/lfs/usr/include'
 /bin/mkdir -p '/mnt/lfs/usr/bin'
 /bin/mkdir -p '/mnt/lfs/usr/share/man/man1'
 /usr/bin/install -c -m 644 src/gnumake.h '/mnt/lfs/usr/include'
  /usr/bin/install -c make '/mnt/lfs/usr/bin'
 /usr/bin/install -c -m 644 doc/make.1 '/mnt/lfs/usr/share/man/man1'
make[2]: Leaving directory '/mnt/lfs/sources/make-4.3'
make[1]: Leaving directory '/mnt/lfs/sources/make-4.3'

real    0m24.137s
user    0m22.830s
sys     0m8.263s
```

各项配置参数的含义如下：

| 参数 | 含义 |
|-----|------|
| --without-guile | 我们是交叉编译， configure 默认会尝试使用来自宿主机的 guile，这会使编译失败，因此直接禁用 guile |

清理安装包

```sh
lfs:/mnt/lfs/sources/make-4.3$ cd ..
lfs:/mnt/lfs/sources$ rm -rf make-4.3
```

### 8.12 安装 Patch-2.7.6

> Patch 包含了通过应用 "补丁" 文件，修改或创建文件的程序，补丁文件通常是 diff 程序创建的。

解压软件包

```sh
lfs:/mnt/lfs/sources$ tar xf patch-2.7.6.tar.xz 
lfs:/mnt/lfs/sources$ cd patch-2.7.6
```

配置并编译安装 (大约耗时40秒)

```sh
time { ./configure --prefix=/usr   \
            --host=$LFS_TGT        \
            --build=$(build-aux/config.guess) && make && make DESTDIR=$LFS install; }

# 安装完成后输出以下信息：
make[4]: Nothing to be done for 'install-data-am'.
make[4]: Leaving directory '/mnt/lfs/sources/patch-2.7.6/lib'
make[3]: Leaving directory '/mnt/lfs/sources/patch-2.7.6/lib'
make[2]: Leaving directory '/mnt/lfs/sources/patch-2.7.6/lib'
Making install in src
make[2]: Entering directory '/mnt/lfs/sources/patch-2.7.6/src'
make[3]: Entering directory '/mnt/lfs/sources/patch-2.7.6/src'
make[3]: Nothing to be done for 'install-data-am'.
 /bin/mkdir -p '/mnt/lfs/usr/bin'
  /usr/bin/install -c patch '/mnt/lfs/usr/bin'
make[3]: Leaving directory '/mnt/lfs/sources/patch-2.7.6/src'
make[2]: Leaving directory '/mnt/lfs/sources/patch-2.7.6/src'
Making install in tests
make[2]: Entering directory '/mnt/lfs/sources/patch-2.7.6/tests'
make[3]: Entering directory '/mnt/lfs/sources/patch-2.7.6/tests'
make[3]: Nothing to be done for 'install-exec-am'.
make[3]: Nothing to be done for 'install-data-am'.
make[3]: Leaving directory '/mnt/lfs/sources/patch-2.7.6/tests'
make[2]: Leaving directory '/mnt/lfs/sources/patch-2.7.6/tests'
make[2]: Entering directory '/mnt/lfs/sources/patch-2.7.6'
make[3]: Entering directory '/mnt/lfs/sources/patch-2.7.6'
make[3]: Nothing to be done for 'install-exec-am'.
 /bin/mkdir -p '/mnt/lfs/usr/share/man/man1'
 /usr/bin/install -c -m 644 'patch.man' '/mnt/lfs/usr/share/man/man1/patch.1'
make[3]: Leaving directory '/mnt/lfs/sources/patch-2.7.6'
make[2]: Leaving directory '/mnt/lfs/sources/patch-2.7.6'
make[1]: Leaving directory '/mnt/lfs/sources/patch-2.7.6'

real    0m49.235s
user    0m37.750s
sys     0m15.345s
```

清理软件包

```sh
lfs:/mnt/lfs/sources/patch-2.7.6$ cd ..
lfs:/mnt/lfs/sources$ rm -rf patch-2.7.6
```

### 8.13 安装 Sed-4.8

> Sed 包含了一个流编辑器

解压软件包

```sh
lfs:/mnt/lfs/sources$ tar xf sed-4.8.tar.xz 
lfs:/mnt/lfs/sources$ cd sed-4.8
```

配置并编译安装 (大约耗时40秒)

```sh
time { ./configure --prefix=/usr   \
            --host=$LFS_TGT        \
            --bindir=/bin && make && make DESTDIR=$LFS install; }

# 安装完成后输出信息：
make[3]: Leaving directory '/mnt/lfs/sources/sed-4.8'
make[2]: Leaving directory '/mnt/lfs/sources/sed-4.8'
Making install in gnulib-tests
make[2]: Entering directory '/mnt/lfs/sources/sed-4.8/gnulib-tests'
make  install-recursive
make[3]: Entering directory '/mnt/lfs/sources/sed-4.8/gnulib-tests'
Making install in .
make[4]: Entering directory '/mnt/lfs/sources/sed-4.8/gnulib-tests'
make[5]: Entering directory '/mnt/lfs/sources/sed-4.8/gnulib-tests'
make[5]: Nothing to be done for 'install-exec-am'.
make[5]: Nothing to be done for 'install-data-am'.
make[5]: Leaving directory '/mnt/lfs/sources/sed-4.8/gnulib-tests'
make[4]: Leaving directory '/mnt/lfs/sources/sed-4.8/gnulib-tests'
make[3]: Leaving directory '/mnt/lfs/sources/sed-4.8/gnulib-tests'
make[2]: Leaving directory '/mnt/lfs/sources/sed-4.8/gnulib-tests'
make[1]: Leaving directory '/mnt/lfs/sources/sed-4.8'

real    0m39.318s
user    0m31.760s
sys     0m12.541s
```

清除软件包

```sh
lfs:/mnt/lfs/sources/sed-4.8$ cd ..
lfs:/mnt/lfs/sources$ rm -rf sed-4.8
```

### 8.14 安装 Tar-1.32

> Tar 软件包提供创建 tar 归档文件，以及对归档文件进行其他操作的功能。Tar 可以对已经创建的归档文件进行提取文件，存储新文件，更新文件，或者列出文件等操作。

解压软件包

```sh
lfs:/mnt/lfs/sources$ tar xf tar-1.32.tar.xz 
lfs:/mnt/lfs/sources$ cd tar-1.32
```

配置并编译安装 (大约耗时1分钟)

```sh
time { ./configure --prefix=/usr              \
            --host=$LFS_TGT                   \
            --build=$(build-aux/config.guess) \
            --bindir=/bin && make && make DESTDIR=$LFS install; }

# 编译完成后输出信息：
make[1]: Leaving directory '/mnt/lfs/sources/tar-1.32/po'
Making install in tests
make[1]: Entering directory '/mnt/lfs/sources/tar-1.32/tests'
make[2]: Entering directory '/mnt/lfs/sources/tar-1.32/tests'
make[2]: Nothing to be done for 'install-exec-am'.
make[2]: Nothing to be done for 'install-data-am'.
make[2]: Leaving directory '/mnt/lfs/sources/tar-1.32/tests'
make[1]: Leaving directory '/mnt/lfs/sources/tar-1.32/tests'
make[1]: Entering directory '/mnt/lfs/sources/tar-1.32'
make[2]: Entering directory '/mnt/lfs/sources/tar-1.32'
make[2]: Nothing to be done for 'install-exec-am'.
make[2]: Nothing to be done for 'install-data-am'.
make[2]: Leaving directory '/mnt/lfs/sources/tar-1.32'
make[1]: Leaving directory '/mnt/lfs/sources/tar-1.32'

real    0m58.117s
user    1m2.212s
sys     0m18.778s
```

清理软件包

```sh
lfs:/mnt/lfs/sources/tar-1.32$ cd ..
lfs:/mnt/lfs/sources$ rm -rf tar-1.32
```

### 8.15 安装 Xz-5.2.5

> Xz 软件包含了文件压缩和解压缩工具，能够处理 lzma 和新的 xz 压缩文件格式。使用 xz 压缩文本文件，可以得到比传统的 gzip 或 bzip2 更好的压缩比。

解压软件包

```sh
lfs:/mnt/lfs/sources$ tar xf xz-5.2.5.tar.xz 
lfs:/mnt/lfs/sources$ cd xz-5.2.5
```

配置并编译安装 (大约耗时35秒)

```sh
time { ./configure --prefix=/usr              \
            --host=$LFS_TGT                   \
            --build=$(build-aux/config.guess) \
            --disable-static                  \
            --docdir=/usr/share/doc/xz-5.2.5 && make && make DESTDIR=$LFS install; }

# 安装完成后输出信息：
make[1]: Leaving directory '/mnt/lfs/sources/xz-5.2.5/po'
Making install in tests
make[1]: Entering directory '/mnt/lfs/sources/xz-5.2.5/tests'
make[2]: Entering directory '/mnt/lfs/sources/xz-5.2.5/tests'
make[2]: Nothing to be done for 'install-exec-am'.
make[2]: Nothing to be done for 'install-data-am'.
make[2]: Leaving directory '/mnt/lfs/sources/xz-5.2.5/tests'
make[1]: Leaving directory '/mnt/lfs/sources/xz-5.2.5/tests'
make[1]: Entering directory '/mnt/lfs/sources/xz-5.2.5'
make[2]: Entering directory '/mnt/lfs/sources/xz-5.2.5'
make[2]: Nothing to be done for 'install-exec-am'.
 /bin/mkdir -p '/mnt/lfs/usr/share/doc/xz-5.2.5/examples'
 /bin/mkdir -p '/mnt/lfs/usr/share/doc/xz-5.2.5'
 /bin/mkdir -p '/mnt/lfs/usr/share/doc/xz-5.2.5/examples_old'
 /usr/bin/install -c -m 644 doc/examples/00_README.txt doc/examples/01_compress_easy.c doc/examples/02_decompress.c doc/examples/03_compress_custom.c doc/examples/04_compress_easy_mt.c doc/examples/Makefile '/mnt/lfs/usr/share/doc/xz-5.2.5/examples'
 /usr/bin/install -c -m 644 doc/examples_old/xz_pipe_comp.c doc/examples_old/xz_pipe_decomp.c '/mnt/lfs/usr/share/doc/xz-5.2.5/examples_old'
 /usr/bin/install -c -m 644 AUTHORS COPYING COPYING.GPLv2 NEWS README THANKS TODO doc/faq.txt doc/history.txt doc/xz-file-format.txt doc/lzma-file-format.txt '/mnt/lfs/usr/share/doc/xz-5.2.5'
make[2]: Leaving directory '/mnt/lfs/sources/xz-5.2.5'
make[1]: Leaving directory '/mnt/lfs/sources/xz-5.2.5'

real    0m35.651s
user    0m40.086s
sys     0m13.146s
```

确保所有基本文件都在正确的目录中

```sh
# 命令1
lfs:/mnt/lfs/sources/xz-5.2.5$ mv -v $LFS/usr/bin/{lzma,unlzma,lzcat,xz,unxz,xzcat} $LFS/bin
renamed '/mnt/lfs/usr/bin/lzma' -> '/mnt/lfs/bin/lzma'
renamed '/mnt/lfs/usr/bin/unlzma' -> '/mnt/lfs/bin/unlzma'
renamed '/mnt/lfs/usr/bin/lzcat' -> '/mnt/lfs/bin/lzcat'
renamed '/mnt/lfs/usr/bin/xz' -> '/mnt/lfs/bin/xz'
renamed '/mnt/lfs/usr/bin/unxz' -> '/mnt/lfs/bin/unxz'
renamed '/mnt/lfs/usr/bin/xzcat' -> '/mnt/lfs/bin/xzcat'
# 命令2
lfs:/mnt/lfs/sources/xz-5.2.5$ mv -v $LFS/usr/lib/liblzma.so.* $LFS/lib
renamed '/mnt/lfs/usr/lib/liblzma.so.5' -> '/mnt/lfs/lib/liblzma.so.5'
renamed '/mnt/lfs/usr/lib/liblzma.so.5.2.5' -> '/mnt/lfs/lib/liblzma.so.5.2.5'
# 命令3
lfs:/mnt/lfs/sources/xz-5.2.5$ ln -svf ../../lib/$(readlink $LFS/usr/lib/liblzma.so) $LFS/usr/lib/liblzma.so
'/mnt/lfs/usr/lib/liblzma.so' -> '../../lib/liblzma.so.5.2.5'
```

清除软件包

```sh
lfs:/mnt/lfs/sources/xz-5.2.5$ cd ..
lfs:/mnt/lfs/sources$ rm -rf xz-5.2.5
```

### 8.16 安装 Binutils-2.35 - Pass 2

> Binutils 包含汇编器、链接器以及其他用于处理目标文件的工具

解压软件包

```sh
lfs:/mnt/lfs/sources$ tar xf binutils-2.35.tar.xz 
lfs:/mnt/lfs/sources$ cd binutils-2.35
```

创建编译目录

```sh
lfs:/mnt/lfs/sources/binutils-2.35$ mkdir -v build
mkdir: created directory 'build'
lfs:/mnt/lfs/sources/binutils-2.35$ cd build/
```

配置并编译安装 (大约耗时4分钟)

```sh
time { ../configure            \
    --prefix=/usr              \
    --build=$(../config.guess) \
    --host=$LFS_TGT            \
    --disable-nls              \
    --enable-shared            \
    --disable-werror           \
    --enable-64-bit-bfd && make && make DESTDIR=$LFS install; }

# 安装完成后输出以下信息：
done
make[5]: Leaving directory '/mnt/lfs/sources/binutils-2.35/build/binutils'
make[4]: Leaving directory '/mnt/lfs/sources/binutils-2.35/build/binutils'
make[3]: Leaving directory '/mnt/lfs/sources/binutils-2.35/build/binutils'
make[2]: Leaving directory '/mnt/lfs/sources/binutils-2.35/build/binutils'
make[1]: Leaving directory '/mnt/lfs/sources/binutils-2.35/build'

real    3m11.903s
user    5m54.872s
sys     1m26.631s
```

各项配置参数的含义如下：

| 参数 | 含义 |
|-----|------|
| --enable-shared | 将 libbfd 构建为共享库 |
| --enable-64-bit-bfd | 启用 64 位支持（在字长较窄的宿主机上）。在 64 位系统上可能不需要，但没有危害。 |

清除软件包

```sh
lfs:/mnt/lfs/sources/binutils-2.35/build$ cd ../..
lfs:/mnt/lfs/sources$ rm -rf binutils-2.35
```

### 8.17 安装 GCC-10.2.0 - Pass 2

> GCC 包含了 GNU 编译器集合，其中有 C 和 C++ 编译器

解压软件包

```sh
lfs:/mnt/lfs/sources$ tar xf gcc-10.2.0.tar.xz 
lfs:/mnt/lfs/sources$ cd gcc-10.2.0
```

正如第一次构建时一样，我们需要将 GMP, MPFR 和 MPC 解压并重命名到指定位置。

```sh
lfs:/mnt/lfs/sources/gcc-10.2.0$ tar -xf ../mpfr-4.1.0.tar.xz
lfs:/mnt/lfs/sources/gcc-10.2.0$ mv -v mpfr-4.1.0 mpfr
renamed 'mpfr-4.1.0' -> 'mpfr'
lfs:/mnt/lfs/sources/gcc-10.2.0$ tar -xf ../gmp-6.2.0.tar.xz
lfs:/mnt/lfs/sources/gcc-10.2.0$ mv -v gmp-6.2.0 gmp
renamed 'gmp-6.2.0' -> 'gmp'
lfs:/mnt/lfs/sources/gcc-10.2.0$ tar -xf ../mpc-1.1.0.tar.gz
lfs:/mnt/lfs/sources/gcc-10.2.0$ mv -v mpc-1.1.0 mpc
renamed 'mpc-1.1.0' -> 'mpc'
```

如果在 `x86_64` 上构建，需要将 64 位库的默认目录名称更改为 `lib`，运行以下代码

```sh
case $(uname -m) in
  x86_64)
    sed -e '/m64=/s/lib64/lib/' -i.orig gcc/config/i386/t-linux64
  ;;
esac
```

创建编译专用目录

```sh
lfs:/mnt/lfs/sources/gcc-10.2.0$ mkdir -v build
mkdir: created directory 'build'
lfs:/mnt/lfs/sources/gcc-10.2.0$ cd build/
```

创建一个符号链接，允许使用 posix 线程支持构建 libgcc

```sh
lfs:/mnt/lfs/sources/gcc-10.2.0/build$ mkdir -pv $LFS_TGT/libgcc
mkdir: created directory 'x86_64-lfs-linux-gnu'
mkdir: created directory 'x86_64-lfs-linux-gnu/libgcc'
lfs:/mnt/lfs/sources/gcc-10.2.0/build$ ln -s ../../../libgcc/gthr-posix.h $LFS_TGT/libgcc/gthr-default.h
```

配置并编译安装 (大约耗时25分钟)

```sh
time { ../configure                \
    --build=$(../config.guess)     \
    --host=$LFS_TGT                \
    --prefix=/usr                  \
    CC_FOR_TARGET=$LFS_TGT-gcc     \
    --with-build-sysroot=$LFS      \
    --enable-initfini-array        \
    --disable-nls                  \
    --disable-multilib             \
    --disable-decimal-float        \
    --disable-libatomic            \
    --disable-libgomp              \
    --disable-libquadmath          \
    --disable-libssp               \
    --disable-libvtv               \
    --disable-libstdcxx            \
    --enable-languages=c,c++ && make && make DESTDIR=$LFS install; }

# 安装完成后输出信息：
make[2]: Leaving directory '/mnt/lfs/sources/gcc-10.2.0/build/gcc'
make[1]: Leaving directory '/mnt/lfs/sources/gcc-10.2.0/build'

real    20m14.550s
user    55m2.283s
sys     6m37.624s
lfs:/mnt/lfs/sources/gcc-10.2.0/build$ 
```

各项配置参数的含义如下：

| 参数 | 含义 |
|-----|------|
| -with-build-sysroot=$LFS | 通常，使用 --host 可确保交叉编译器用于构建 GCC，并且该编译器知道它必须在 $LFS 中查找头文件和库。但是 GCC 的构建系统使用了其他工具，这些工具并不知道这个位置。需要此配置项才能让他们在 $LFS 中而不是在宿主机上找到所需的文件。 |
| --enable-initfini-array | 在 x86 上使用本地编译器构建本地编译器时会自动启用此选项。但是在这里，我们使用交叉编译器进行构建，因此我们需要显式设置此选项。 |

创建符号链接：许多程序和脚本运行 cc 而不是 gcc，后者可保持程序的通用性，因此可用于并非总是安装 GNU C 编译器的各种 UNIX 系统。运行以下链接命令，使系统管理员可以自由决定安装哪个 C 编译器。

```sh
lfs:/mnt/lfs/sources/gcc-10.2.0/build$ ln -sv gcc $LFS/usr/bin/cc
'/mnt/lfs/usr/bin/cc' -> 'gcc'
```

清除软件包

```sh
lfs:/mnt/lfs/sources/gcc-10.2.0/build$ cd ../..
lfs:/mnt/lfs/sources$ rm -rf gcc-10.2.0
```

## 9. 进入 Chroot 并构建额外的临时工具

### 9.1 概述

本章节将展示如何构建临时系统的最后缺失位：首先安装构建各种软件包所依赖的工具，之后会有三个软件包用于运行测试。随后，所有循环依赖都将被解决，我们就可以使用 `chroot` 环境，从而完全隔离(除了正在运行的内核)用于构建的宿主机操作系统。

为了使隔离环境能够正常运行，必须与正在运行的宿主机内核建立一些通信。这是通过所谓的 *虚拟内核文件系统* 实现的，该文件系统必须在进入 chroot 环境时被挂载。可以通过 `findmnt` 命令检查它是否被挂载。

直到接下来的 [第9.4-进入Chroot环境](###9.4 进入Chroot环境) 为止，本章节的命令都必须由 `root` 用户执行，同时要保证 `LFS` 环境变量被正确设置。进入 Chroot 环境之后，所有的命令都将被 root 执行，但幸运的是，chroot 环境下的 root 指令没有访问构建 LFS 的系统(即宿主机系统)的权限。但无论如何您都要小心，因为某些错误格式的指令还是会轻易地毁掉整个 LFS 系统。

### 9.2 更改所有权

注意：接下来的命令都要求在 root 用户，而不是 lfs 用户下执行。同时，再次强调，请检查一下 root 环境下是否正确配置了 `$LFS` 环境变量。

现在，`$LFS`中的所有目录层次结构都由用户 lfs 所拥有，但 lfs 用户是个仅存在于宿主机系统上的用户。如果 $LFS 下的目录和文件都保持现状，那它们都将被一个没有相应账户信息的用户ID所拥有。这样是很危险的，因为以后创建的用户有可能获得相同的用户ID，从而会获得 $LFS 下所有文件的所有权，这会使文件暴露在恶意的操作中。

要解决此问题，需要运行一下指令，将 $LFS 目录的所有权更改为 root 用户：

```sh
# 先切换到root用户
lfs:~$ su
Password: 
root@ubuntu:/home/lfs#
# 更改文件所有权
chown -R root:root $LFS/{usr,lib,var,etc,bin,sbin,tools}
case $(uname -m) in
  x86_64) chown -R root:root $LFS/lib64 ;;
esac
```

### 9.3 准备虚拟内核文件系统

内核导出的各种文件系统用于与内核本身进行通信。这些文件系统是虚拟的，因为它们没有使用磁盘空间。文件系统的内容驻留在内存中。

首先创建用于挂载文件系统的目录

```sh
root@ubuntu:~# mkdir -pv $LFS/{dev,proc,sys,run}
mkdir: created directory '/mnt/lfs/dev'
mkdir: created directory '/mnt/lfs/proc'
mkdir: created directory '/mnt/lfs/sys'
mkdir: created directory '/mnt/lfs/run'
```

#### 9.3.1 创建初始设备节点

当内核引导系统时，需要存在一些设备节点，特别是控制台设备(console)和空设备(null)。必须在硬盘上创建设备节点，以便它们在内核填充 `/dev` 之前可用，另外在使用 `init=/bin/bash` 启动 Linux 时也是如此。通过以下命令能够创建节点。

```sh
root@ubuntu:~# mknod -m 600 $LFS/dev/console c 5 1
root@ubuntu:~# mknod -m 666 $LFS/dev/null c 1 3
```

#### 9.3.2 挂载和填充 /dev

使用设备填充 `/dev` 目录的推荐方法是在 `/dev` 目录上挂载一个虚拟文件系统（例如 tmpfs），并允许在检测或访问设备时在该虚拟文件系统上动态创建设备。设备创建一般在启动过程中由 `Udev` 完成。由于这个新系统还没有 Udev 并且还没有启动，所以需要手动挂载和填充 /dev。这是通过绑定挂载主机系统的 /dev 目录来完成的。绑定挂载是一种特殊类型的挂载，它允许您创建目录或挂载到其他位置的镜像。使用以下命令来实现此目的：

```sh
root@ubuntu:~# mount -v --bind /dev $LFS/dev
mount: /dev bound on /mnt/lfs/dev.
```

#### 9.3.3 挂载虚拟内核文件系统

现在挂载剩余的虚拟内核文件系统：

```sh
root@ubuntu:~# mount -v --bind /dev/pts $LFS/dev/pts
mount: /dev/pts bound on /mnt/lfs/dev/pts.
root@ubuntu:~# mount -vt proc proc $LFS/proc
mount: proc mounted on /mnt/lfs/proc.
root@ubuntu:~# mount -vt sysfs sysfs $LFS/sys
mount: sysfs mounted on /mnt/lfs/sys.
root@ubuntu:~# mount -vt tmpfs tmpfs $LFS/run
mount: tmpfs mounted on /mnt/lfs/run.
```

在某些宿主机系统中，`/dev/shm` 是 `/run/shm` 的符号链接。`/run tmpfs` 挂载在它上面，所以在这种情况下只需要创建一个目录，输入以下指令：

```sh
if [ -h $LFS/dev/shm ]; then
  mkdir -pv $LFS/$(readlink $LFS/dev/shm)
fi
```

### 9.4 进入 Chroot 环境

现在，构建其余所需工具的所有软件包都已在系统上，是时候进入 chroot 环境来完成其余临时工具的安装了。该环境也将用于安装最终系统。以 root 用户身份运行以下命令，进入目前仅填充了临时工具的环境：

```sh
chroot "$LFS" /usr/bin/env -i   \
    HOME=/root                  \
    TERM="$TERM"                \
    PS1='(lfs chroot) \u:\w\$ ' \
    PATH=/bin:/usr/bin:/sbin:/usr/sbin \
    /bin/bash --login +h
```

运行上述命令的输出信息如下：

```sh
root@ubuntu:~# chroot "$LFS" /usr/bin/env -i   \
>     HOME=/root                  \
>     TERM="$TERM"                \
>     PS1='(lfs chroot) \u:\w\$ ' \
>     PATH=/bin:/usr/bin:/sbin:/usr/sbin \
>     /bin/bash --login +h
(lfs chroot) I have no name!:/# 
```

上述命令中，`env`命令的`-i`选项会清除 chroot 环境的所有变量。因此，就只有上述命令中所提到的 HOME、TERM、PS1 和 PATH 变量会被设置。`TERM=$TERM` 命令会将 chroot 内部的 TERM 变量设置为与 chroot 外部相同的值，因为 vim 和 less 这样的程序需要这个变量才能正常运行。如果需要其他变量，例如 CFLAGS 或 CXXFLAGS，可以由此设置。

从现在开始，不再需要使用 `$LFS` 变量，因为所有工作都将仅限于 LFS 文件系统。**这是因为 Bash shell 被告知 $LFS 现在是根 (/) 目录。**

请注意 `/tools/bin` 不在 `PATH` 中。这意味着在 chroot 环境中将不能再使用交叉工具链。还需要保证 shell 不'记忆"执行过的程序位置，因此需要传递 +h 参数给 bash 以关闭散列功能。

请注意，bash 提示`I have no name!`，这是正常的，因为尚未创建 `/etc/passwd` 文件。

注意：本章剩余部分和后续章节中的所有命令都在 chroot 环境中运行，这一点很重要。如火你因为重启或者其他原因导致离开了这个环境，请确保先进行 [9.3.2 挂载和填充 /dev](####9.3.2 挂载和填充 /dev) 和 [9.3.3 挂载虚拟内核文件系统](####9.3.3 挂载虚拟内核文件系统) ，并且进入 chroot 环境之后，再继续进行之后的其他操作。

### 9.5 创建目录

是时候在 LFS 文件系统中创建完整的目录结构了。

通过执行以下命令，创建一些前几章不需要的根级目录：

注意：下面的一些目录早先已经进行了提前说明或在安装一些软件包时已被创建。为了完整起见，它们会在下面重复。

```sh
(lfs chroot) I have no name!:/# mkdir -pv /{boot,home,mnt,opt,srv}
mkdir: created directory '/boot'
mkdir: created directory '/home'
mkdir: created directory '/mnt'
mkdir: created directory '/opt'
mkdir: created directory '/srv'
```

通过执行以下命令，在根目录以下创建所需的子目录集：

```sh
(lfs chroot) I have no name!:/# mkdir -pv /etc/{opt,sysconfig}
mkdir: created directory '/etc/opt'
mkdir: created directory '/etc/sysconfig'
(lfs chroot) I have no name!:/# mkdir -pv /lib/firmware
mkdir: created directory '/lib/firmware'
(lfs chroot) I have no name!:/# mkdir -pv /media/{floppy,cdrom}
mkdir: created directory '/media'
mkdir: created directory '/media/floppy'
mkdir: created directory '/media/cdrom'
(lfs chroot) I have no name!:/# mkdir -pv /usr/{,local/}{bin,include,lib,sbin,src}
mkdir: created directory '/usr/src'
mkdir: created directory '/usr/local'
mkdir: created directory '/usr/local/bin'
mkdir: created directory '/usr/local/include'
mkdir: created directory '/usr/local/lib'
mkdir: created directory '/usr/local/sbin'
mkdir: created directory '/usr/local/src'
(lfs chroot) I have no name!:/# mkdir -pv /usr/{,local/}share/{color,dict,doc,info,locale,man}
mkdir: created directory '/usr/share/color'
mkdir: created directory '/usr/share/dict'
mkdir: created directory '/usr/local/share'
mkdir: created directory '/usr/local/share/color'
mkdir: created directory '/usr/local/share/dict'
mkdir: created directory '/usr/local/share/doc'
mkdir: created directory '/usr/local/share/info'
mkdir: created directory '/usr/local/share/locale'
mkdir: created directory '/usr/local/share/man'
(lfs chroot) I have no name!:/# mkdir -pv /usr/{,local/}share/{misc,terminfo,zoneinfo}
mkdir: created directory '/usr/share/zoneinfo'
mkdir: created directory '/usr/local/share/misc'
mkdir: created directory '/usr/local/share/terminfo'
mkdir: created directory '/usr/local/share/zoneinfo'
(lfs chroot) I have no name!:/# mkdir -pv /usr/{,local/}share/man/man{1..8}
mkdir: created directory '/usr/share/man/man2'
mkdir: created directory '/usr/share/man/man6'
mkdir: created directory '/usr/local/share/man/man1'
mkdir: created directory '/usr/local/share/man/man2'
mkdir: created directory '/usr/local/share/man/man3'
mkdir: created directory '/usr/local/share/man/man4'
mkdir: created directory '/usr/local/share/man/man5'
mkdir: created directory '/usr/local/share/man/man6'
mkdir: created directory '/usr/local/share/man/man7'
mkdir: created directory '/usr/local/share/man/man8'
(lfs chroot) I have no name!:/# mkdir -pv /var/{cache,local,log,mail,opt,spool}
mkdir: created directory '/var/cache'
mkdir: created directory '/var/local'
mkdir: created directory '/var/log'
mkdir: created directory '/var/mail'
mkdir: created directory '/var/opt'
mkdir: created directory '/var/spool'
(lfs chroot) I have no name!:/# mkdir -pv /var/lib/{color,misc,locate}
mkdir: created directory '/var/lib/color'
mkdir: created directory '/var/lib/misc'
mkdir: created directory '/var/lib/locate'
(lfs chroot) I have no name!:/# ln -sfv /run /var/run
'/var/run' -> '/run'
(lfs chroot) I have no name!:/# ln -sfv /run/lock /var/lock
'/var/lock' -> '/run/lock'
(lfs chroot) I have no name!:/# install -dv -m 0750 /root
install: creating directory '/root'
(lfs chroot) I have no name!:/# install -dv -m 1777 /tmp /var/tmp
install: creating directory '/tmp'
install: creating directory '/var/tmp'
(lfs chroot) I have no name!:/# 
```

默认情况下，目录是使用 755 权限模式创建的，但这并不是所有目录都需要的。在上面的命令中，进行了两次更改,一个更改为 root 用户的主目录，另一个更改为临时文件的目录。

第一个模式更改确保不是任何人都可以进入 /root 目录——就像普通用户进入他或她的主目录一样。第二个模式更改确保任何用户都可以写入 /tmp 和 /var/tmp 目录，但不能从中删除其他用户的文件。后者被所谓的“粘滞位”禁止，即 1777 位掩码中的最高位(1)。

#### 9.5.1 FHS 规范说明

目录树基于文件系统层次结构标准 (FHS)(可从 https://refspecs.linuxfoundation.org/fhs.shtml 获得)。FHS 还指定了一些目录的可选存在，例如 `/usr/local/games` 和 `/usr/share/games`。我们只创建需要的目录。但是，您可以随意创建这些目录。

### 9.6 创建基本文件和符号链接

历史上，Linux 会在 /etc/mtab 文件中维护一个已挂载文件系统的列表。现代内核则在内部维护此列表，并通过 `/proc` 文件系统将其公开给用户。为了满足那些需要 /etc/mtab 的实用程序，创建以下符号链接：

```sh
(lfs chroot) I have no name!:/# ln -sv /proc/self/mounts /etc/mtab
'/etc/mtab' -> '/proc/self/mounts'
```

创建一个基本的 /etc/hosts 文件，以便在某些测试套件以及 Perl 的配置文件之一中引用：

```sh
(lfs chroot) I have no name!:/# echo "127.0.0.1 localhost $(hostname)" > /etc/hosts
```

为了让 root 用户能够登录并识别名称 `root`，`/etc/passwd` 和 `/etc/group` 文件中必须有相关条目。

通过运行以下命令创建 `/etc/passwd` 文件：

```sh
cat > /etc/passwd << "EOF"
root:x:0:0:root:/root:/bin/bash
bin:x:1:1:bin:/dev/null:/bin/false
daemon:x:6:6:Daemon User:/dev/null:/bin/false
messagebus:x:18:18:D-Bus Message Daemon User:/var/run/dbus:/bin/false
nobody:x:99:99:Unprivileged User:/dev/null:/bin/false
EOF
```

root 的实际密码将在稍后设置。

通过运行以下命令创建 `/etc/group` 文件：

```sh
cat > /etc/group << "EOF"
root:x:0:
bin:x:1:daemon
sys:x:2:
kmem:x:3:
tape:x:4:
tty:x:5:
daemon:x:6:
floppy:x:7:
disk:x:8:
lp:x:9:
dialout:x:10:
audio:x:11:
video:x:12:
utmp:x:13:
usb:x:14:
cdrom:x:15:
adm:x:16:
messagebus:x:18:
input:x:24:
mail:x:34:
kvm:x:61:
wheel:x:97:
nogroup:x:99:
users:x:999:
EOF
```

以上所创建的组不是任何标准的一部分——它们中的一部分是由第 9 章中 Udev 配置的要求决定的，另一部分是由许多现有 Linux 发行版采用的通用约定决定的。此外，一些测试套件依赖于特定的用户或组。Linux 标准基础（LSB，可从 http://refspecs.linuxfoundation.org/lsb.shtml 获得）仅建议，组根的组ID (GID) 为 0 的之外，bin组的组ID 为 1 。系统管理员可以自由选择所有其他组名和 GID，因为编写良好的程序不依赖于 GID 编号，而是使用组名。

下一章中的某些测试需要普通用户。我们在此处添加此用户并在该章末尾删除此帐户。

```sh
(lfs chroot) I have no name!:/# echo "tester:x:$(ls -n $(tty) | cut -d" " -f3):101::/home/tester:/bin/bash" >> /etc/passwd
(lfs chroot) I have no name!:/# echo "tester:x:101:" >> /etc/group
(lfs chroot) I have no name!:/# install -o tester -d /home/tester
```

若要移除 `I have no name!` 提示，可以启动一个新的shell。由于 `/etc/passwd` 和 `/etc/group` 文件已经创建，用户名和组名解析现在已经可以工作：

```sh
(lfs chroot) I have no name!:/# exec /bin/bash --login +h
(lfs chroot) root:/# 
```

请注意 +h 指令的使用。这告诉 bash 不要使用其内部路径散列。如果没有这个指令，bash 会记住它已执行的二进制文件的路径。为确保新编译的二进制文件安装后立即使用，+h 指令将在本章和下一章中使用。

login、agetty 和 init 程序（和其他程序）使用许多日志文件来记录信息，例如谁登录到系统以及何时登录。但是，如果这些程序不存在，它们将不会写入日志文件。初始化日志文件并赋予它们适当的权限：

```sh
(lfs chroot) root:/# touch /var/log/{btmp,lastlog,faillog,wtmp}
(lfs chroot) root:/# chgrp -v utmp /var/log/lastlog
changed group of '/var/log/lastlog' from root to utmp
(lfs chroot) root:/# chmod -v 664  /var/log/lastlog
mode of '/var/log/lastlog' changed from 0644 (rw-r--r--) to 0664 (rw-rw-r--)
(lfs chroot) root:/# chmod -v 600  /var/log/btmp
mode of '/var/log/btmp' changed from 0644 (rw-r--r--) to 0600 (rw-------)
```

`/var/log/wtmp` 文件记录所有登录和注销。 `/var/log/lastlog` 文件记录每个用户上次登录的时间。`/var/log/faillog` 文件记录失败的登录尝试。 `/var/log/btmp` 文件记录了错误的登录尝试。

注意：`/run/utmp` 文件记录了当前登录的用户。这个文件是在启动脚本中动态创建的。

### 9.7 安装 Libstdc++ from GCC-10.2.0, Pass 2

在构建 gcc-pass2 时，我们不得不推迟 C++ 标准库的安装，因为没有合适的编译器可以编译它。我们无法使用该部分中内置的编译器，因为它是本地编译器，不应在 chroot 之外使用，并且存在使用某些宿主机组件污染库的风险。

注意：Libstdc++ 是 GCC 源代码的一部分。您应该首先解压 GCC tarball 并切换到 gcc-10.2.0 目录。

```sh
(lfs chroot) root:/# ls
bin   dev  home  lib64       media  opt   root  sbin     srv  tmp    usr
boot  etc  lib   lost+found  mnt    proc  run   sources  sys  tools  var
(lfs chroot) root:/# cd sources/
(lfs chroot) root:/sources# tar xf gcc-10.2.0.tar.xz 
(lfs chroot) root:/sources# cd gcc-10.2.0
(lfs chroot) root:/sources/gcc-10.2.0# 
```

创建在 gcc 树中构建 libstdc++ 时存在的链接：

```sh
(lfs chroot) root:/sources/gcc-10.2.0# ln -s gthr-posix.h libgcc/gthr-default.h
```

为 libstdc++ 创建一个单独的构建目录并进入该目录：

```sh
(lfs chroot) root:/sources/gcc-10.2.0# mkdir -v build
mkdir: created directory 'build'
(lfs chroot) root:/sources/gcc-10.2.0# cd build/
```

配置并编译安装 (大约耗时4分钟)

```sh
time { ../libstdc++-v3/configure     \
    CXXFLAGS="-g -O2 -D_GNU_SOURCE"  \
    --prefix=/usr                    \
    --disable-multilib               \
    --disable-nls                    \
    --host=$(uname -m)-lfs-linux-gnu \
    --disable-libstdcxx-pch && make && make install; }

# 安装完成后输出以下信息：
make[2]: Nothing to be done for 'install-data-am'.
make[2]: Leaving directory '/sources/gcc-10.2.0/build'
make[1]: Leaving directory '/sources/gcc-10.2.0/build'

real    3m46.487s
user    3m4.398s
sys     0m29.506s
```

各项配置参数的含义如下：

| 参数 | 含义 |
|-----|------|
| CXXFLAGS="-g -O2 -D_GNU_SOURCE"  | 在进行 GCC 的完整构建时，这些标志由顶层 Makefile 传递。 |
| --host=$(uname -m)-lfs-linux-gnu | 我们需要模拟如果这个包是作为完整编译器构建的一部分构建的会发生什么。此配置项将通过 GCC 的构建机制传递给 configure |
| --disable-libstdcxx-pch  | 阻止安装现阶段不需要的预编译包含文件。 |

清除安装包

```sh
(lfs chroot) root:/sources/gcc-10.2.0/build# cd ../..
(lfs chroot) root:/sources# rm -rf gcc-10.2.0
```

### 9.8 安装 Gettext-0.21

Gettext 包含用于国际化和本地化的实用程序。这些允许使用 NLS（本机语言支持）编译程序，使它们能够以用户的母语输出消息。

对于我们的临时工具集，我们只需要从 Gettext 安装三个程序。

解压软件包

```sh
(lfs chroot) root:/sources# tar xf gettext-0.21.tar.xz 
(lfs chroot) root:/sources# cd gettext-0.21
```

配置并编译 (大约耗时8分钟)

```sh
time { ./configure --disable-shared && make; }

# 编译完成后输出以下信息：
make[6]: Leaving directory '/sources/gettext-0.21/gettext-tools/gnulib-tests'
make[5]: Leaving directory '/sources/gettext-0.21/gettext-tools/gnulib-tests'
make[4]: Leaving directory '/sources/gettext-0.21/gettext-tools/gnulib-tests'
Making all in examples
make[4]: Entering directory '/sources/gettext-0.21/gettext-tools/examples'
Making all in po
make[5]: Entering directory '/sources/gettext-0.21/gettext-tools/examples/po'
make[5]: Nothing to be done for 'all'.
make[5]: Leaving directory '/sources/gettext-0.21/gettext-tools/examples/po'
make[5]: Entering directory '/sources/gettext-0.21/gettext-tools/examples'
make[5]: Nothing to be done for 'all-am'.
make[5]: Leaving directory '/sources/gettext-0.21/gettext-tools/examples'
make[4]: Leaving directory '/sources/gettext-0.21/gettext-tools/examples'
Making all in doc
make[4]: Entering directory '/sources/gettext-0.21/gettext-tools/doc'
make[4]: Nothing to be done for 'all'.
make[4]: Leaving directory '/sources/gettext-0.21/gettext-tools/doc'
make[4]: Entering directory '/sources/gettext-0.21/gettext-tools'
make[4]: Leaving directory '/sources/gettext-0.21/gettext-tools'
make[3]: Leaving directory '/sources/gettext-0.21/gettext-tools'
make[2]: Leaving directory '/sources/gettext-0.21/gettext-tools'
make[2]: Entering directory '/sources/gettext-0.21'
make[2]: Nothing to be done for 'all-am'.
make[2]: Leaving directory '/sources/gettext-0.21'
make[1]: Leaving directory '/sources/gettext-0.21'

real    8m18.989s
user    6m15.913s
sys     1m28.522s
```

安装 msgfmt, msgmerge 和 xgettext 这三个程序：

```sh
(lfs chroot) root:/sources/gettext-0.21# cp -v gettext-tools/src/{msgfmt,msgmerge,xgettext} /usr/bin
'gettext-tools/src/msgfmt' -> '/usr/bin/msgfmt'
'gettext-tools/src/msgmerge' -> '/usr/bin/msgmerge'
'gettext-tools/src/xgettext' -> '/usr/bin/xgettext'
```

清除安装包

```sh
(lfs chroot) root:/sources/gettext-0.21# cd ..
(lfs chroot) root:/sources# rm -rf gettext-0.21
```

### 9.9 安装 Bison-3.7.1

Bison 包含一个解析器生成器

解压软件包

```sh
(lfs chroot) root:/sources# tar xf bison-3.7.1.tar.xz 
(lfs chroot) root:/sources# cd bison-3.7.1
```

配置并编译安装 (大约耗时2分钟)

```sh
time { ./configure --prefix=/usr \
            --docdir=/usr/share/doc/bison-3.7.1 && make && make install; }

# 安装完成后输出信息：
make[3]: Leaving directory '/sources/bison-3.7.1'
make[2]: Leaving directory '/sources/bison-3.7.1'
make[1]: Leaving directory '/sources/bison-3.7.1'

real    1m23.505s
user    1m3.631s
sys     0m13.651s
```

清除软件包

```sh
(lfs chroot) root:/sources/bison-3.7.1# cd ..
(lfs chroot) root:/sources# rm -rf bison-3.7.1
```

### 9.10 安装 Perl-5.32.0

Perl 包含实用报表提取语言

解压软件包

```sh
(lfs chroot) root:/sources# tar xf perl-5.32.0.tar.xz 
(lfs chroot) root:/sources# cd perl-5.32.0
```

配置并编译安装 (大约耗时8分钟)

```sh
time { sh Configure -des                                 \
             -Dprefix=/usr                               \
             -Dvendorprefix=/usr                         \
             -Dprivlib=/usr/lib/perl5/5.32/core_perl     \
             -Darchlib=/usr/lib/perl5/5.32/core_perl     \
             -Dsitelib=/usr/lib/perl5/5.32/site_perl     \
             -Dsitearch=/usr/lib/perl5/5.32/site_perl    \
             -Dvendorlib=/usr/lib/perl5/5.32/vendor_perl \
             -Dvendorarch=/usr/lib/perl5/5.32/vendor_perl && make && make install; }

# 安装完成后输出以下信息：
./perl -Ilib -I. installman --destdir= 
Manual page installation was disabled by Configure

real    7m6.965s
user    6m9.540s
sys     0m45.596s
```

清除安装包

```sh
(lfs chroot) root:/sources/perl-5.32.0# cd ..
(lfs chroot) root:/sources# rm -rf perl-5.32.0
```

### 9.11 安装 Python-3.8.5

Python 3 包含 Python 开发环境。它对于面向对象编程、编写脚本、大型程序原型或开发整个应用程序非常有用。

注意：有两个包文件的名称以 python 开头。从中提取的是 Python-3.8.5.tar.xz（注意大写的第一个字母）。

解压软件包

```sh
(lfs chroot) root:/sources# tar xf Python-3.8.5.tar.xz 
(lfs chroot) root:/sources# cd Python-3.8.5
```

配置并编译安装 (大约耗时5分钟)

```sh
time { ./configure --prefix=/usr   \
            --enable-shared \
            --without-ensurepip && make && make install; }

# 安装完成后输出以下信息：
if test "xno" != "xno"  ; then \
        case no in \
                upgrade) ensurepip="--upgrade" ;; \
                install|*) ensurepip="" ;; \
        esac; \
        LD_LIBRARY_PATH=/sources/Python-3.8.5 ./python -E -m ensurepip \
                $ensurepip --root=/ ; \
fi

real    5m19.581s
user    4m51.537s
sys     0m38.621s
```

清除软件包

```sh
(lfs chroot) root:/sources/Python-3.8.5# cd ..
(lfs chroot) root:/sources# rm -rf Python-3.8.5
```

### 9.12 安装 Texinfo-6.7

Texinfo 包含用于读取、写入和转换信息页面的程序。

解压软件包

```sh
(lfs chroot) root:/sources# tar xf texinfo-6.7.tar.xz 
(lfs chroot) root:/sources# cd texinfo-6.7
```

配置并编译安装 (大约耗时1分钟)

注意：在配置过程中，进行了一项指示 `TestXS_la-TestXS.lo` 错误的测试。这与 LFS 无关，应该被忽略。

```sh
time { ./configure --prefix=/usr && make && make install; }

# 安装完成后输出以下信息：
make[2]: Leaving directory '/sources/texinfo-6.7/man'
make[1]: Leaving directory '/sources/texinfo-6.7/man'
make[1]: Entering directory '/sources/texinfo-6.7'
make[2]: Entering directory '/sources/texinfo-6.7'
make[2]: Nothing to be done for 'install-exec-am'.
make[2]: Nothing to be done for 'install-data-am'.
make[2]: Leaving directory '/sources/texinfo-6.7'
make[1]: Leaving directory '/sources/texinfo-6.7'

real    1m15.022s
user    0m56.700s
sys     0m13.983s
```

清除软件包

```sh
(lfs chroot) root:/sources/texinfo-6.7# cd ..
(lfs chroot) root:/sources# rm -rf texinfo-6.7
```

### 9.13 安装 Util-linux-2.36

Util-linux 包含各种实用程序

解压软件包

```sh
(lfs chroot) root:/sources# tar xf util-linux-2.36.tar.xz 
(lfs chroot) root:/sources# cd util-linux-2.36
```

创建一个目录为 hwclock 程序启用存储：

```sh
(lfs chroot) root:/sources/util-linux-2.36# mkdir -pv /var/lib/hwclock
mkdir: created directory '/var/lib/hwclock'
```

配置并编译安装 (大约耗时3分钟)

```sh
time { ./configure ADJTIME_PATH=/var/lib/hwclock/adjtime    \
            --docdir=/usr/share/doc/util-linux-2.36 \
            --disable-chfn-chsh  \
            --disable-login      \
            --disable-nologin    \
            --disable-su         \
            --disable-setpriv    \
            --disable-runuser    \
            --disable-pylibmount \
            --disable-static     \
            --without-python && make && make install; }

# 安装完成后输出以下信息：
make  install-data-hook
make[4]: Entering directory '/sources/util-linux-2.36'
make[4]: Nothing to be done for 'install-data-hook'.
make[4]: Leaving directory '/sources/util-linux-2.36'
make[3]: Leaving directory '/sources/util-linux-2.36'
make[2]: Leaving directory '/sources/util-linux-2.36'
make[1]: Leaving directory '/sources/util-linux-2.36'

real    3m27.898s
user    2m45.419s
sys     0m34.177s
```

清除软件包

```sh
(lfs chroot) root:/sources/util-linux-2.36# cd ..
(lfs chroot) root:/sources# rm -rf util-linux-2.36
```

### 9.14 清理和保存临时系统

`.la` 文件仅在与静态库链接时有用。在使用动态共享库时，尤其是在使用非自动工具构建系统时，它们是不需要的，并且可能有害。在 chroot 环境下删除这些文件：

```sh
(lfs chroot) root:/sources# find /usr/{lib,libexec} -name \*.la -delete
```

删除临时工具的文档，以防止它们出现在最终系统中，这样可以节省大约 35 MB：

```sh
(lfs chroot) root:/sources# rm -rf /usr/share/{info,man,doc}/*
```

**本节中的所有其余步骤都是可选的。**然而，一旦您开始安装第 10 章中的软件包，临时工具就会被覆盖。因此，如下所述备份临时工具可能是个好主意。仅当您的磁盘空间非常短缺时才需要执行其他步骤。

**注意：建议现在使用虚拟机快照进行备份。**

以下步骤是从 chroot 环境外部执行的。这意味着，您必须先离开 chroot 环境才能继续。这样做的原因是：

* 确保对象在操作时未被使用。
* 访问 chroot 环境之外的文件系统位置，以存储/读取出于安全原因不应放置在 $LFS 层次结构中的备份存档。

离开 chroot 环境并卸载内核虚拟文件系统：

注意：以下所有指令均由 root 执行。请格外注意您将要运行的命令，因为此处的错误可能会修改您的宿主机系统。请注意，默认情况下为用户 lfs 设置了环境变量 LFS，但可能未为 root 设置。每当命令由 root 执行时，**请确保您已相应地设置 LFS**。

```sh
# 警告：请先使用虚拟机快照功能，备份当前状态！
(lfs chroot) root:/sources# exit    
logout
root@ubuntu:~# umount $LFS/dev{/pts,}
root@ubuntu:~# umount $LFS/{sys,proc,run}
root@ubuntu:~# 
```

#### 9.14.1 清理

如果 LFS 分区相当小，了解可以删除的不必要的项目是很有帮助的。目前为止构建的可执行文件和库中，包含了略多于 90 MB 的不需要的调试符号。

从二进制文件中去除调试符号：

```sh
# 首先检查是否设置 LFS 环境变量
root@ubuntu:~# echo $LFS         
/mnt/lfs
# 在 root 下执行以下三条命令
# 命令1
root@ubuntu:~# strip --strip-debug $LFS/usr/lib/*
strip: Warning: '/mnt/lfs/usr/lib/audit' is a directory
strip: Warning: '/mnt/lfs/usr/lib/bash' is a directory
strip: Warning: '/mnt/lfs/usr/lib/gawk' is a directory
strip: Warning: '/mnt/lfs/usr/lib/gcc' is a directory
strip: Warning: '/mnt/lfs/usr/lib/gconv' is a directory
strip:/mnt/lfs/usr/lib/libc.so: File format not recognized
strip:/mnt/lfs/usr/lib/libgcc_s.so: File format not recognized
strip:/mnt/lfs/usr/lib/libm.a: File format not recognized
strip:/mnt/lfs/usr/lib/libm.so: File format not recognized
strip:/mnt/lfs/usr/lib/libncurses.so: File format not recognized
strip:/mnt/lfs/usr/lib/libstdc++.so.6.0.28-gdb.py: File format not recognized
strip: Warning: '/mnt/lfs/usr/lib/perl5' is a directory
strip: Warning: '/mnt/lfs/usr/lib/pkgconfig' is a directory
strip: Warning: '/mnt/lfs/usr/lib/python3.8' is a directory
strip: Warning: '/mnt/lfs/usr/lib/terminfo' is a directory
strip: Warning: '/mnt/lfs/usr/lib/texinfo' is a directory
# 命令2
root@ubuntu:~# strip --strip-unneeded $LFS/usr/{,s}bin/*
strip:/mnt/lfs/usr/bin/2to3: File format not recognized
strip:/mnt/lfs/usr/bin/2to3-3.8: File format not recognized
strip:/mnt/lfs/usr/bin/bashbug: File format not recognized
strip:/mnt/lfs/usr/bin/catchsegv: File format not recognized
strip:/mnt/lfs/usr/bin/corelist: File format not recognized
strip:/mnt/lfs/usr/bin/cpan: File format not recognized
strip:/mnt/lfs/usr/bin/enc2xs: File format not recognized
strip:/mnt/lfs/usr/bin/encguess: File format not recognized
strip:/mnt/lfs/usr/bin/gunzip: File format not recognized
strip:/mnt/lfs/usr/bin/gzexe: File format not recognized
strip:/mnt/lfs/usr/bin/h2ph: File format not recognized
strip:/mnt/lfs/usr/bin/h2xs: File format not recognized
strip:/mnt/lfs/usr/bin/idle3: File format not recognized
strip:/mnt/lfs/usr/bin/idle3.8: File format not recognized
strip:/mnt/lfs/usr/bin/instmodsh: File format not recognized
strip:/mnt/lfs/usr/bin/json_pp: File format not recognized
strip:/mnt/lfs/usr/bin/ldd: File format not recognized
strip:/mnt/lfs/usr/bin/libnetcfg: File format not recognized
strip:/mnt/lfs/usr/bin/lzcmp: File format not recognized
strip:/mnt/lfs/usr/bin/lzdiff: File format not recognized
strip:/mnt/lfs/usr/bin/lzegrep: File format not recognized
strip:/mnt/lfs/usr/bin/lzfgrep: File format not recognized
strip:/mnt/lfs/usr/bin/lzgrep: File format not recognized
strip:/mnt/lfs/usr/bin/lzless: File format not recognized
strip:/mnt/lfs/usr/bin/lzmore: File format not recognized
strip:/mnt/lfs/usr/bin/makeinfo: File format not recognized
strip:/mnt/lfs/usr/bin/mtrace: File format not recognized
strip:/mnt/lfs/usr/bin/ncursesw6-config: File format not recognized
strip:/mnt/lfs/usr/bin/pdftexi2dvi: File format not recognized
strip:/mnt/lfs/usr/bin/perlbug: File format not recognized
strip:/mnt/lfs/usr/bin/perldoc: File format not recognized
strip:/mnt/lfs/usr/bin/perlivp: File format not recognized
strip:/mnt/lfs/usr/bin/perlthanks: File format not recognized
strip:/mnt/lfs/usr/bin/piconv: File format not recognized
strip:/mnt/lfs/usr/bin/pl2pm: File format not recognized
strip:/mnt/lfs/usr/bin/pod2html: File format not recognized
strip:/mnt/lfs/usr/bin/pod2man: File format not recognized
strip:/mnt/lfs/usr/bin/pod2texi: File format not recognized
strip:/mnt/lfs/usr/bin/pod2text: File format not recognized
strip:/mnt/lfs/usr/bin/pod2usage: File format not recognized
strip:/mnt/lfs/usr/bin/podchecker: File format not recognized
strip:/mnt/lfs/usr/bin/prove: File format not recognized
strip:/mnt/lfs/usr/bin/ptar: File format not recognized
strip:/mnt/lfs/usr/bin/ptardiff: File format not recognized
strip:/mnt/lfs/usr/bin/ptargrep: File format not recognized
strip:/mnt/lfs/usr/bin/pydoc3: File format not recognized
strip:/mnt/lfs/usr/bin/pydoc3.8: File format not recognized
strip:/mnt/lfs/usr/bin/python3-config: File format not recognized
strip:/mnt/lfs/usr/bin/python3.8-config: File format not recognized
strip:/mnt/lfs/usr/bin/shasum: File format not recognized
strip:/mnt/lfs/usr/bin/sotruss: File format not recognized
strip:/mnt/lfs/usr/bin/splain: File format not recognized
strip:/mnt/lfs/usr/bin/streamzip: File format not recognized
strip:/mnt/lfs/usr/bin/texi2any: File format not recognized
strip:/mnt/lfs/usr/bin/texi2dvi: File format not recognized
strip:/mnt/lfs/usr/bin/texi2pdf: File format not recognized
strip:/mnt/lfs/usr/bin/texindex: File format not recognized
strip:/mnt/lfs/usr/bin/tzselect: File format not recognized
strip:/mnt/lfs/usr/bin/uncompress: File format not recognized
strip:/mnt/lfs/usr/bin/updatedb: File format not recognized
strip:/mnt/lfs/usr/bin/xsubpp: File format not recognized
strip:/mnt/lfs/usr/bin/xtrace: File format not recognized
strip:/mnt/lfs/usr/bin/xzcmp: File format not recognized
strip:/mnt/lfs/usr/bin/xzdiff: File format not recognized
strip:/mnt/lfs/usr/bin/xzegrep: File format not recognized
strip:/mnt/lfs/usr/bin/xzfgrep: File format not recognized
strip:/mnt/lfs/usr/bin/xzgrep: File format not recognized
strip:/mnt/lfs/usr/bin/xzless: File format not recognized
strip:/mnt/lfs/usr/bin/xzmore: File format not recognized
strip:/mnt/lfs/usr/bin/yacc: File format not recognized
strip:/mnt/lfs/usr/bin/zcat: File format not recognized
strip:/mnt/lfs/usr/bin/zcmp: File format not recognized
strip:/mnt/lfs/usr/bin/zdiff: File format not recognized
strip:/mnt/lfs/usr/bin/zegrep: File format not recognized
strip:/mnt/lfs/usr/bin/zfgrep: File format not recognized
strip:/mnt/lfs/usr/bin/zforce: File format not recognized
strip:/mnt/lfs/usr/bin/zgrep: File format not recognized
strip:/mnt/lfs/usr/bin/zipdetails: File format not recognized
strip:/mnt/lfs/usr/bin/zless: File format not recognized
strip:/mnt/lfs/usr/bin/zmore: File format not recognized
strip:/mnt/lfs/usr/bin/znew: File format not recognized
# 命令3
root@ubuntu:~# strip --strip-unneeded $LFS/tools/bin/*
root@ubuntu:~# 
```

这些命令将跳过一些它无法识别其文件格式的文件。其中大部分是脚本而不是二进制文件。

此时，您应该在 chroot 分区上至少有 5GB 的可用空间，可用于在下一阶段构建和安装 Glibc 和 GCC。如果您可以构建和安装 Glibc，那么您也可以构建和安装其余的。您可以使用命令 `df -h $LFS` 检查可用磁盘空间。

```sh
root@ubuntu:~# df -h $LFS
Filesystem      Size  Used Avail Use% Mounted on
/dev/sdb1        20G  2.4G   17G  13% /mnt/lfs
```

#### 9.14.2 备份

现在已经创建了基本工具，是时候考虑备份了。当先前构建的包中的每个检查都成功通过时，您的临时工具处于良好状态，可能会被备份以供以后重用。如果在后续章节中出现致命故障，通常会证明删除所有内容并重新开始（更仔细地）是恢复的最佳选择。不幸的是，所有临时工具也将被删除。为避免花费额外的时间来重做已成功构建的内容，请准备一个备份。

确保在用户 root 的主目录中至少有 600 MB 的可用磁盘空间（软件源包文件 tarball 将包含在备份存档中）。

通过运行以下命令创建备份存档：

```sh
cd $LFS && tar -cJpf $HOME/lfs-temp-tools-10.0.tar.xz .
```

文件归档的时间较长 (大概耗时20分钟)，可以使用以下命令查看当前压缩包的大小：

```sh
# 备份完成后文件大小为796MB
root@ubuntu:~# ls -lh /root/lfs-temp-tools-10.0.tar.xz
-rw-r--r-- 1 root root 796M Dec 21 22:33 /root/lfs-temp-tools-10.0.tar.xz
```

#### 9.14.3 恢复

如果出现了一些错误需要重新开始，您可以使用此备份来恢复临时工具并节省一些恢复时间。由于源位于 $LFS 下，它们也包含在备份存档中，因此无需再次下载。**检查 $LFS 设置正确后**，通过执行以下命令恢复备份：

```sh
cd $LFS && rm -rf ./* && tar -xpf $HOME/lfs-temp-tools-10.0.tar.xz
```

再次仔细检查环境是否已正确设置，继续构建系统的其余部分。

**!! 重要：如果您离开了 chroot 环境以去除调试符号、创建备份或使用恢复重新开始构建，请记住现在再次挂载内核虚拟文件系统，如第 [9.3 准备虚拟内核文件系统](### 9.3 准备虚拟内核文件系统) 中重新进入 chroot 环境（参见第 [9.4 进入 Chroot 环境](### 9.4 进入 Chroot 环境)），然后再继续。**

```sh
# 回到 chroot 环境的例程
zhj@ubuntu:~$ echo $LFS
# 现在是在宿主机普通用户环境下，这里已经没有 LFS 环境变量了
zhj@ubuntu:~$ export LFS=/mnt/lfs
zhj@ubuntu:~$ sudo mkdir -pv $LFS
[sudo] password for zhj: 
zhj@ubuntu:~$ sudo mount -v -t ext4 /dev/sdb1 $LFS
mount: /dev/sdb1 mounted on /mnt/lfs.
# 切换到 root 用户
zhj@ubuntu:~$ su 
Password: 
root@ubuntu:/home/zhj# cd
root@ubuntu:~# mkdir -pv $LFS/{dev,proc,sys,run}
root@ubuntu:~# mknod -m 600 $LFS/dev/console c 5 1
mknod: /mnt/lfs/dev/console: File exists
root@ubuntu:~# mknod -m 666 $LFS/dev/null c 1 3
mknod: /mnt/lfs/dev/null: File exists
root@ubuntu:~# mount -v --bind /dev $LFS/dev
mount: /dev bound on /mnt/lfs/dev.
root@ubuntu:~# mount -v --bind /dev/pts $LFS/dev/pts
mount: /dev/pts bound on /mnt/lfs/dev/pts.
root@ubuntu:~# mount -vt proc proc $LFS/proc
mount: proc mounted on /mnt/lfs/proc.
root@ubuntu:~# mount -vt sysfs sysfs $LFS/sys
mount: sysfs mounted on /mnt/lfs/sys.
root@ubuntu:~# mount -vt tmpfs tmpfs $LFS/run
mount: tmpfs mounted on /mnt/lfs/run.
# 执行以下命令：
if [ -h $LFS/dev/shm ]; then
   mkdir -pv $LFS/$(readlink $LFS/dev/shm)
fi
# 执行以下命令即可进入 chroot 环境
chroot "$LFS" /usr/bin/env -i   \
    HOME=/root                  \
    TERM="$TERM"                \
    PS1='(lfs chroot) \u:\w\$ ' \
    PATH=/bin:/usr/bin:/sbin:/usr/sbin \
    /bin/bash --login +h
# 注意： chroot 环境下，不再需要 LFS 环境变量！
```

## 10. 安装基本系统软件

### 10.1 概述

在本章中，我们开始正式构建 LFS 系统。

软件的安装过程很简单。尽管在许多情况下，安装说明可以变得更短和更通用，但我们选择为每个包提供完整的说明，以尽量减少出错的可能性。了解 Linux 系统工作原理的关键是了解每个包的用途以及您（或系统）可能需要它的原因。

我们不建议使用优化(We do not recommend using optimizations.)。它们可以使程序运行得稍微快一些，但它们也可能在运行程序时造成编译困难和问题。如果一个包在使用优化时拒绝编译，请尝试在不优化的情况下编译它，看看是否能解决问题。即使在使用优化时包确实编译了，由于代码和构建工具之间的复杂交互，它也存在编译不正确的风险。另请注意，书中未指定值的 -march 和 -mtune 选项尚未经过测试。这可能会导致工具链包（Binutils、GCC 和 Glibc）出现问题。使用编译器优化所获得的微小潜在收益通常被风险所抵消。鼓励 LFS 的首次构建者在没有自定义优化的情况下进行构建。后续系统仍然会运行得非常快，同时也很稳定。

在安装说明之前，每个安装页面都提供了有关软件包的信息，包括对它所包含的内容、构建大约需要多长时间以及在此构建过程中需要多少磁盘空间的简要描述。按照安装说明，有一个包安装的程序和库（连同简要说明）的列表。

注意：SBU 值和所需的磁盘空间包括第 8 章中所有适用包的测试套件数据。SBU 值是使用单个 CPU 内核 (-j1) 计算的所有操作。

#### 8.1.1. 关于库

通常，LFS 编辑器不鼓励构建和安装静态库。大多数静态库的最初用途在现代 Linux 系统中已经过时。此外，将静态库链接到程序可能是有害的。如果需要更新库以消除安全问题，则所有使用静态库的程序都需要重新链接到新库。由于静态库的使用并不总是很明显，因此甚至可能不知道相关程序（以及进行链接所需的过程）。

在本章的过程中，我们删除或禁用了大多数静态库的安装。通常这是通过传递一个 `--disable-static` 选项来配置的。在其他情况下，需要替代方法。在少数情况下，尤其是 glibc 和 gcc，静态库的使用对于一般包构建过程仍然必不可少。

### 8.2 软件包管理

包管理是 LFS 手册中经常被要求的补充。包管理器允许跟踪文件的安装，从而轻松删除和升级包。除了二进制文件和库文件，包管理器将处理配置文件的安装。在您开始怀疑之前，不——本节不会讨论或推荐任何特定的包管理器。它提供的是更流行的技术及其工作方式的综述。最适合您的包管理器可能是这些技术之一，也可能是这些技术中的两个或多个的组合。本节简要提及升级软件包时可能出现的问题。

在 LFS 或 BLFS 中没有提到包管理器的一些原因包括：

* 处理包管理会将重点从这些书的目标上移开——教如何构建 Linux 系统。
* 包管理有多种解决方案，每种都有其优点和缺点。一个让所有观众都满意的作品是很困难的。

有一些关于包管理主题的提示。访问提示项目，看看其中一个是否适合您的需要。

#### 8.2.1 升级问题

包管理器可以在新版本发布时轻松升级到新版本。通常可以使用 LFS 和 BLFS 书籍中的说明升级到较新版本。以下是升级软件包时应注意的一些要点，尤其是在正在运行的系统上。

* 如果 Glibc 需要升级到更新的版本（例如从 glibc-2.31 升级到 glibc-2.32），重建 LFS 会更安全。尽管您可以按依赖顺序重建所有包，但我们不建议这样做。
* 如果更新包含共享库的包，并且库的名称发生更改，则需要重新编译动态链接到该库的任何包，以便链接到较新的库。（请注意，包版本和库名称之间没有关联。）例如，考虑安装名为 libfoo.so.1 的共享库的包 foo-1.2.3。如果您将软件包升级到安装名为 libfoo.so.2 的共享库的较新版本 foo-1.2.4。在这种情况下，任何动态链接到 libfoo.so.1 的包都需要重新编译以链接到 libfoo.so.2 才能使用新的库版本。除非重新编译所有依赖包，否则不应删除以前的库。

#### 8.2.2 包管理技术

以下是一些常见的包管理技术。在决定包管理器之前，对各种技术做一些研究，特别是特定方案的缺点。

##### 8.2.2.1 这一切都在我的脑海中！

是的，这是一种包管理技术。有些人认为不需要包管理器，因为他们非常了解包，并且知道每个包安装了哪些文件。一些用户也不需要任何包管理，因为他们计划在包更改时重建整个系统。

##### 8.2.2.2 在单独的目录中安装

这是一个简单的包管理，不需要任何额外的包来管理安装。例如，软件包 foo-1.1 安装在 /usr/pkg/foo-1.1 中，并且符号链接从 /usr/pkg/foo 到 /usr/pkg/foo-1.1。安装新版本 foo-1.2 时，它安装在 /usr/pkg/foo-1.2 中，之前的符号链接被新版本的符号链接替换。

PATH、LD_LIBRARY_PATH、MANPATH、INFOPATH 和 CPPFLAGS 等环境变量需要扩展以包含 /usr/pkg/foo。对于多个包，该方案变得难以管理。

##### 8.2.2.3 符号链接样式包管理

这是先前包管理技术的一种变体。每个包的安装类似于以前的方案。但是不是创建符号链接，而是每个文件都被符号链接到 /usr 层次结构中。这消除了扩展环境变量的需要。尽管用户可以创建符号链接以自动创建，但许多包管理器已使用这种方法编写。一些流行的包括 Stow、Epkg、Graft 和 Depot。

安装需要伪造，以便包认为它安装在 /usr 中，但实际上它安装在 /usr/pkg 层次结构中。以这种方式安装通常不是一项微不足道的任务。例如，假设您正在安装一个软件包 libfoo-1.1。以下说明可能无法正确安装软件包：

```sh
./configure --prefix=/usr/pkg/libfoo/1.1
make
make install
```

安装将起作用，但依赖包可能不会如您所愿链接到 libfoo。如果你编译一个链接到 libfoo 的包，你可能会注意到它链接到 /usr/pkg/libfoo/1.1/lib/libfoo.so.1 而不是你期望的 /usr/lib/libfoo.so.1。正确的方法是使用 DESTDIR 策略来伪造包的安装。这种方法的工作原理如下：

```sh
./configure --prefix=/usr
make
make DESTDIR=/usr/pkg/libfoo/1.1 install
```

大多数软件包支持这种方法，但也有一些不支持。对于不合规的包，您可能需要手动安装该包，或者您可能会发现将一些有问题的包安装到 /opt 中会更容易。

##### 8.2.2.4 基于时间戳

在这种技术中，文件在安装包之前加上时间戳。安装后，使用相应选项的简单使用查找命令可以生成创建时间戳文件后安装的所有文件的日志。用这种方法编写的包管理器是 install-log。

虽然这种方案具有简单的优点，但它有两个缺点。如果在安装过程中使用当前时间以外的任何时间戳安装文件，则包管理器将不会跟踪这些文件。此外，此方案只能在一次安装一个包时使用。如果两个软件包安装在两个不同的控制台上，则日志不可靠。

##### 8.2.2.5 跟踪安装脚本

在这种方法中，安装脚本执行的命令被记录下来。可以使用两种技术：

LD_PRELOAD 环境变量可以设置为指向安装前要预加载的库。在安装过程中，该库通过将自身附加到各种可执行文件（例如 cp、install、mv）并跟踪修改文件系统的系统调用来跟踪正在安装的包。为了使这种方法起作用，所有可执行文件都需要在没有 suid 或 sgid 位的情况下动态链接。在安装过程中预加载库可能会导致一些不需要的副作用。因此，建议执行一些测试以确保包管理器不会破坏任何内容并记录所有适当的文件。

第二种技术是使用 strace，它记录在安装脚本执行期间进行的所有系统调用。

##### 8.2.2.6 创建包档案

在这个方案中，包安装被伪造到一个单独的树中，如 Symlink 风格的包管理中所述。安装后，将使用已安装的文件创建包存档。然后，此存档用于在本地计算机上安装软件包，甚至可以用于在其他计算机上安装软件包。

大多数商业发行版中的包管理器都使用这种方法。遵循这种方法的包管理器的例子有 RPM（顺便说一句，Linux 标准基本规范要求）、pkg-utils、Debian 的 apt 和 Gentoo 的 Portage 系统。描述如何为 LFS 系统采用这种包管理风格的提示位于 http://www.linuxfromscratch.org/hints/downloads/files/fakeroot.txt

**创建包含依赖信息的包文件很复杂，超出了 LFS 的范围。**

Slackware 使用基于 tar 的系统进行包归档。该系统故意不像更复杂的包管理器那样处理包依赖关系。有关 Slackware 包管理的详细信息，请参阅 http://www.slackbook.org/html/package-management.html。

##### 8.2.2.7 基于用户的管理

该方案是 LFS 独有的，由 Matthias Benkmann 设计，可从 Hints Project 获得。在此方案中，每个包都作为单独的用户安装到标准位置。通过检查用户 ID 可以轻松识别属于包的文件。这种方法的特点和缺点过于复杂，无法在本节中描述。有关详细信息，请参阅 http://www.linuxfromscratch.org/hints/downloads/files/more_control_and_pkg_man.txt 中的提示。

#### 8.2.3 在多个系统上部署 LFS

LFS 系统的优点之一是不存在依赖于磁盘系统上文件的位置。将 LFS 构建克隆到另一台与基本系统具有相同架构的计算机上就像在包含根目录的 LFS 分区上使用 tar 一样简单，（对于基本 LFS 构建，未压缩大约 250MB），通过网络传输或 CD-ROM 将该文件复制到新系统并扩展它。从那时起，必须更改一些配置文件。可能需要更新的配置文件包括：/etc/hosts、/etc/fstab、/etc/passwd、/etc/group、/etc/shadow、/etc/ld.so.conf、/etc/sysconfig/rc .site、/etc/sysconfig/network 和 /etc/sysconfig/ifconfig.eth0。根据系统硬件和原始内核配置的差异，可能需要为新系统构建自定义内核。

注意：在类似但不相同的体系结构之间复制时，有一些问题报告。例如，Intel 系统的指令集与 AMD 处理器不同，某些处理器的更高版本可能具有早期版本中不可用的指令。

**最后，必须通过第 10.4 节“使用 GRUB 设置启动过程”使新系统可启动。**

### 安装 Man-pages-5.08

Man-pages 包含了 2,200 多个手册页

解压软件包

```sh
(lfs chroot) root:/# cd sources/
(lfs chroot) root:/sources# tar xf man-pages-5.08.tar.xz
(lfs chroot) root:/sources# cd man-pages-5.08
```

安装软件包

```sh
(lfs chroot) root:/sources/man-pages-5.08# make install 
for i in man?; do \
        install -d -m 755 /usr/share/man/"$i" || exit $?; \
        install -m 644 "$i"/* /usr/share/man/"$i" || exit $?; \
done
(lfs chroot) root:/sources/man-pages-5.08# 
```

清除软件包

```sh
(lfs chroot) root:/sources/man-pages-5.08# cd ..
(lfs chroot) root:/sources# rm -rf man-pages-5.08
```

### 8.4 安装 Tcl-8.6.10

Tcl 包含工具命令语言，这是一种强大的通用脚本语言。 Expect 包是用 Tcl 语言编写的。

安装这个包和接下来的两个（Expect 和 DejaGNU）是为了支持运行 GCC 和 binutils 和其他包的测试套件。安装三个用于测试目的的包似乎有些过分，但知道最重要的工具都在正常工作，即使不是必需的，也是非常令人放心的。运行本章中的测试套件需要这些包。

解压软件包

```sh
(lfs chroot) root:/sources# tar xf tcl8.6.10-src.tar.gz 
(lfs chroot) root:/sources# cd tcl8.6.10 
```

解压文档包

```sh
(lfs chroot) root:/sources/tcl8.6.10# tar -xf ../tcl8.6.10-html.tar.gz --strip-components=1
```

准备配置文件

```sh
(lfs chroot) root:/sources/tcl8.6.10# SRCDIR=$(pwd)
(lfs chroot) root:/sources/tcl8.6.10# cd unix
# 继续输入以下配置指令：
./configure --prefix=/usr           \
            --mandir=/usr/share/man \
            $([ "$(uname -m)" = x86_64 ] && echo --enable-64bit)

# 配置完成后输出以下信息：
configure: creating ./config.status
config.status: creating Makefile
config.status: creating dltest/Makefile
config.status: creating tclConfig.sh
config.status: creating tcl.pc
(lfs chroot) root:/sources/tcl8.6.10/unix# 
```

构建软件包

```sh
# 分别输入以下指令：

make

sed -e "s|$SRCDIR/unix|/usr/lib|" \
    -e "s|$SRCDIR|/usr/include|"  \
    -i tclConfig.sh

sed -e "s|$SRCDIR/unix/pkgs/tdbc1.1.1|/usr/lib/tdbc1.1.1|" \
    -e "s|$SRCDIR/pkgs/tdbc1.1.1/generic|/usr/include|"    \
    -e "s|$SRCDIR/pkgs/tdbc1.1.1/library|/usr/lib/tcl8.6|" \
    -e "s|$SRCDIR/pkgs/tdbc1.1.1|/usr/include|"            \
    -i pkgs/tdbc1.1.1/tdbcConfig.sh

sed -e "s|$SRCDIR/unix/pkgs/itcl4.2.0|/usr/lib/itcl4.2.0|" \
    -e "s|$SRCDIR/pkgs/itcl4.2.0/generic|/usr/include|"    \
    -e "s|$SRCDIR/pkgs/itcl4.2.0|/usr/include|"            \
    -i pkgs/itcl4.2.0/itclConfig.sh

unset SRCDIR
```

make 命令之后的各种 sed 指令用于从配置文件中删除对构建目录的引用，并将它们替换为安装目录。这对于 LFS 的其余部分不是强制性的，但在以后构建的包使用 Tcl 的情况下可能需要。

若要测试结果，请执行：

```sh
(lfs chroot) root:/sources/tcl8.6.10/unix# make test

# 输出结果：
Tests ended at Tue Dec 21 15:52:04 +0000 2021
all.tcl:        Total   137     Passed  121     Skipped 16      Failed  0
Sourced 0 Test Files.
Number of tests skipped for each constraint:
        8       have_gdbm
        8       have_lmdb
make[1]: Leaving directory '/sources/tcl8.6.10/unix/pkgs/thread2.8.5'
```

注意：在测试结果中有几个与 clock.test 相关的地方表明失败，但最后的总结表明没有失败。 clock.test 通过一个完整的 LFS 系统。

安装软件包：

```sh
(lfs chroot) root:/sources/tcl8.6.10/unix# make install
```

使已安装的库可写，以便稍后可以删除调试符号：

```sh
(lfs chroot) root:/sources/tcl8.6.10/unix# chmod -v u+w /usr/lib/libtcl8.6.so
mode of '/usr/lib/libtcl8.6.so' changed from 0555 (r-xr-xr-x) to 0755 (rwxr-xr-x)
```

安装 Tcl 的头文件。下一个包 Expect 需要它们。

```sh
(lfs chroot) root:/sources/tcl8.6.10/unix# make install-private-headers
Installing private header files to /usr/include/
```

现在创建一个必要的符号链接:

```sh
(lfs chroot) root:/sources/tcl8.6.10/unix# ln -sfv tclsh8.6 /usr/bin/tclsh
'/usr/bin/tclsh' -> 'tclsh8.6'
```

### 8.5 安装 Expect-5.45.4

Expect 包含用于通过脚本对话自动执行交互式应用程序的工具，例如 telnet、ftp、passwd、fsck、rlogin 和 tip。Expect 也可用于测试这些相同的应用程序，以及减轻使用其他任何东西都非常困难的各种任务。 DejaGnu 框架是用Expect 编写的。

安装软件包

```sh
(lfs chroot) root:/sources# tar xf expect5.45.4.tar.gz 
(lfs chroot) root:/sources# cd expect5.45.4
```

准备配置文件

```sh
./configure --prefix=/usr           \
            --with-tcl=/usr/lib     \
            --enable-shared         \
            --mandir=/usr/share/man \
            --with-tclinclude=/usr/include

# 配置完成后输出以下信息：
checking for sys/param.h... yes
configure: creating ./config.status
config.status: creating Makefile
(lfs chroot) root:/sources/expect5.45.4# 
```

构建安装包并测试

```sh
(lfs chroot) root:/sources/expect5.45.4# time { make && make test; }

# 完成后输出以下信息：
all.tcl:        Total   29      Passed  29      Skipped 0       Failed  0
Sourced 0 Test Files.

real    0m21.503s
user    0m7.499s
sys     0m0.789s
```

安装软件包

```sh
(lfs chroot) root:/sources/expect5.45.4# make install
(lfs chroot) root:/sources/expect5.45.4# ln -svf expect5.45.4/libexpect5.45.4.so /usr/lib
'/usr/lib/libexpect5.45.4.so' -> 'expect5.45.4/libexpect5.45.4.so'
```

### 8.6 安装 DejaGNU-1.6.2

DejaGnu 包包含一个在 GNU 工具上运行测试套件的框架。它是用expect编写的，它本身使用Tcl（工具命令语言）。

解压软件包

```sh
(lfs chroot) root:/sources# tar xf dejagnu-1.6.2.tar.gz 
(lfs chroot) root:/sources# cd dejagnu-1.6.2
```

准备配置文件

```sh
(lfs chroot) root:/sources/dejagnu-1.6.2# ./configure --prefix=/usr
(lfs chroot) root:/sources/dejagnu-1.6.2# makeinfo --html --no-split -o doc/dejagnu.html doc/dejagnu.texi
(lfs chroot) root:/sources/dejagnu-1.6.2# makeinfo --plaintext       -o doc/dejagnu.txt  doc/dejagnu.texi
```

构建并安装软件包

```sh
(lfs chroot) root:/sources/dejagnu-1.6.2# make install
(lfs chroot) root:/sources/dejagnu-1.6.2# install -v -dm755  /usr/share/doc/dejagnu-1.6.2
install: creating directory '/usr/share/doc/dejagnu-1.6.2'
(lfs chroot) root:/sources/dejagnu-1.6.2# install -v -m644   doc/dejagnu.{html,txt} /usr/share/doc/dejagnu-1.6.2
'doc/dejagnu.html' -> '/usr/share/doc/dejagnu-1.6.2/dejagnu.html'
'doc/dejagnu.txt' -> '/usr/share/doc/dejagnu-1.6.2/dejagnu.txt'
```

若要测试，输入以下指令：

```sh
(lfs chroot) root:/sources/dejagnu-1.6.2# make check

# 输出以下信息：
                === runtest Summary ===

# of expected passes            77
DejaGnu version 1.6.2
Expect version  5.45.4
Tcl version     8.6

make[1]: Leaving directory '/sources/dejagnu-1.6.2'
```

清除软件包，这里同时清除 Tcl、Expect 和 DejaGNU 三个软件的软件包：

```sh
(lfs chroot) root:/sources# rm -rf tcl8.6.10 expect5.45.4 dejagnu-1.6.2
```

### 8.7 安装 Iana-Etc-20200821

Iana-Etc 为网络服务和协议提供数据

解压软件包

```sh
(lfs chroot) root:/sources# tar xf iana-etc-20200821.tar.gz 
(lfs chroot) root:/sources# cd iana-etc-20200821
```

对于这个包，我们只需要将文件复制到位：

```sh
(lfs chroot) root:/sources/iana-etc-20200821# cp services protocols /etc
```

删除软件包

```sh
(lfs chroot) root:/sources/iana-etc-20200821# cd ..
(lfs chroot) root:/sources# rm -rf iana-etc-20200821
```

### 8.8 安装 Glibc-2.32

Glibc 包包含主要的 C 库。该库提供了用于分配内存、搜索目录、打开和关闭文件、读取和写入文件、字符串处理、模式匹配、算术等的基本例程。

#### 8.8.1 配置安装 Glibc

解压软件包

```sh
(lfs chroot) root:/sources# tar xf glibc-2.32.tar.xz 
(lfs chroot) root:/sources# cd glibc-2.32
```

一些 Glibc 程序使用非 FHS 兼容的 /var/db 目录来存储它们的运行时数据。应用以下补丁，使此类程序将其运行时数据存储在符合 FHS 的位置：

```sh
(lfs chroot) root:/sources/glibc-2.32# patch -Np1 -i ../glibc-2.32-fhs-1.patch
patching file Makeconfig
Hunk #1 succeeded at 245 (offset -5 lines).
patching file nscd/nscd.h
Hunk #1 succeeded at 161 (offset 49 lines).
patching file nss/db-Makefile
patching file sysdeps/generic/paths.h
patching file sysdeps/unix/sysv/linux/paths.h
```

Glibc 文档建议在专用构建目录中构建 Glibc：

```sh
(lfs chroot) root:/sources/glibc-2.32# mkdir -v build
mkdir: created directory 'build'
(lfs chroot) root:/sources/glibc-2.32# cd build/
```

配置并编译 (大约耗时20分钟)

```sh
time { ../configure --prefix=/usr                \
             --disable-werror                    \
             --enable-kernel=3.2                 \
             --enable-stack-protector=strong     \
             --with-headers=/usr/include         \
             libc_cv_slibdir=/lib && make; }

# 完成后输出以下内容：
make[2]: Leaving directory '/sources/glibc-2.32/elf'
make[1]: Leaving directory '/sources/glibc-2.32'

real    17m20.008s
user    13m29.272s
sys     3m15.147s
```

！重要：在本节中，Glibc 的测试套件被认为是至关重要的。在任何情况下都不要跳过它。

一般有几个测试没有通过。下面列出的测试失败通常可以安全地忽略。

```sh
case $(uname -m) in
  i?86)   ln -sfnv $PWD/elf/ld-linux.so.2        /lib ;;
  x86_64) ln -sfnv $PWD/elf/ld-linux-x86-64.so.2 /lib ;;
esac

# 输出信息：
(lfs chroot) root:/sources/glibc-2.32/build# case $(uname -m) in
>   i?86)   ln -sfnv $PWD/elf/ld-linux.so.2        /lib ;;
>   x86_64) ln -sfnv $PWD/elf/ld-linux-x86-64.so.2 /lib ;;
> esac
'/lib/ld-linux-x86-64.so.2' -> '/sources/glibc-2.32/build/elf/ld-linux-x86-64.so.2'
```

注意：在 chroot 环境中构建的这个阶段需要上面的符号链接来运行测试。它将在下面的安装阶段被覆盖。

```sh
# 以下命令大约耗时55分钟
(lfs chroot) root:/sources/glibc-2.32/build# time { make check; }

# 运行结束后输出以下信息：
UNSUPPORTED: nptl/test-mutexattr-printers
UNSUPPORTED: nptl/test-rwlock-printers
UNSUPPORTED: nptl/test-rwlockattr-printers
FAIL: nptl/tst-mutex10
UNSUPPORTED: nptl/tst-pthread-getattr
UNSUPPORTED: nss/tst-nss-db-endgrent
UNSUPPORTED: nss/tst-nss-db-endpwent
UNSUPPORTED: nss/tst-nss-files-hosts-long
UNSUPPORTED: nss/tst-nss-test3
UNSUPPORTED: resolv/tst-resolv-ai_idn
UNSUPPORTED: resolv/tst-resolv-ai_idn-latin1
UNSUPPORTED: stdlib/tst-system
UNSUPPORTED: string/tst-strerror
UNSUPPORTED: string/tst-strsignal
Summary of test results:
      3 FAIL
   4233 PASS
     28 UNSUPPORTED
     17 XFAIL
      2 XPASS
make[1]: *** [Makefile:633: tests] Error 1
make[1]: Leaving directory '/sources/glibc-2.32'
make: *** [Makefile:9: check] Error 2
(lfs chroot) root:/sources/glibc-2.32/build# 
```

您可能会看到一些测试失败。 Glibc 测试套件在某种程度上依赖于宿主机系统。这是某些 LFS 版本中最常见的问题列表：

* 已知 io/tst-lchmod 在 LFS chroot 环境中失败。
* 已知 misc/tst-ttyname 在 LFS chroot 环境中失败。
* 已知nss/tst-nss-files-hosts-multi 测试可能由于尚未确定的原因而失败。
* rt/tst-cputimer{1,2,3} 测试取决于宿主机系统内核。已知内核 4.14.91–4.14.96、4.19.13–4.19.18 和 4.20.0–4.20.5 会导致这些测试失败。
* 在 CPU 不是相对较新的 Intel 或 AMD 处理器的系统上运行时，math 测试有时会失败。

虽然这是一个无害的消息，但 Glibc 的安装阶段会抱怨 /etc/ld.so.conf 的缺失。使用以下方法防止此警告：

```sh
(lfs chroot) root:/sources/glibc-2.32/build# touch /etc/ld.so.conf
```

修复生成的 Makefile 以跳过在 LFS 部分环境中失败的不需要的健全性检查：

```sh
(lfs chroot) root:/sources/glibc-2.32/build# sed '/test-installation/s@$(PERL)@echo not running@' -i ../Makefile
```

安装软件包 (大约耗时1分钟)

```sh
(lfs chroot) root:/sources/glibc-2.32/build# time { make install; }

# 运行结束后输出以下信息：
LD_SO=ld-linux-x86-64.so.2 CC="gcc" echo not running scripts/test-installation.pl /sources/glibc-2.32/build/
not running scripts/test-installation.pl /sources/glibc-2.32/build/
make[1]: Leaving directory '/sources/glibc-2.32'

real    1m22.972s
user    1m5.824s
sys     0m13.157s
```

安装 nscd 的配置文件和运行时目录：

```sh
(lfs chroot) root:/sources/glibc-2.32/build# cp -v ../nscd/nscd.conf /etc/nscd.conf
'../nscd/nscd.conf' -> '/etc/nscd.conf'
(lfs chroot) root:/sources/glibc-2.32/build# mkdir -pv /var/cache/nscd
mkdir: created directory '/var/cache/nscd'
```

接下来，安装可以使系统以不同语言响应的语言环境。不需要任何语言环境，但如果缺少某些语言环境，未来软件包的测试套件将跳过重要的测试用例。

可以使用 localedef 程序安装各个语言环境。例如，下面的第一个 localedef 命令将与 /usr/share/i18n/locales/cs_CZ 字符集无关的语言环境定义与 /usr/share/i18n/charmaps/UTF-8.gz 字符集定义结合起来，并将结果附加到 /usr /lib/locale/locale-archive 文件。以下说明将安装最佳测试覆盖率所需的最小语言环境集：

```sh
(lfs chroot) root:/sources/glibc-2.32/build# mkdir -pv /usr/lib/locale
mkdir: created directory '/usr/lib/locale'
(lfs chroot) root:/sources/glibc-2.32/build# localedef -i POSIX -f UTF-8 C.UTF-8 2> /dev/null || true
(lfs chroot) root:/sources/glibc-2.32/build# localedef -i cs_CZ -f UTF-8 cs_CZ.UTF-8
(lfs chroot) root:/sources/glibc-2.32/build# localedef -i de_DE -f ISO-8859-1 de_DE
(lfs chroot) root:/sources/glibc-2.32/build# localedef -i de_DE@euro -f ISO-8859-15 de_DE@euro
(lfs chroot) root:/sources/glibc-2.32/build# localedef -i de_DE -f UTF-8 de_DE.UTF-8
(lfs chroot) root:/sources/glibc-2.32/build# localedef -i el_GR -f ISO-8859-7 el_GR
(lfs chroot) root:/sources/glibc-2.32/build# localedef -i en_GB -f UTF-8 en_GB.UTF-8
(lfs chroot) root:/sources/glibc-2.32/build# localedef -i en_HK -f ISO-8859-1 en_HK
(lfs chroot) root:/sources/glibc-2.32/build# localedef -i en_PH -f ISO-8859-1 en_PH
(lfs chroot) root:/sources/glibc-2.32/build# localedef -i en_US -f ISO-8859-1 en_US
(lfs chroot) root:/sources/glibc-2.32/build# localedef -i en_US -f UTF-8 en_US.UTF-8
(lfs chroot) root:/sources/glibc-2.32/build# localedef -i es_MX -f ISO-8859-1 es_MX
(lfs chroot) root:/sources/glibc-2.32/build# localedef -i fa_IR -f UTF-8 fa_IR
(lfs chroot) root:/sources/glibc-2.32/build# localedef -i fr_FR -f ISO-8859-1 fr_FR
(lfs chroot) root:/sources/glibc-2.32/build# localedef -i fr_FR@euro -f ISO-8859-15 fr_FR@euro
(lfs chroot) root:/sources/glibc-2.32/build# localedef -i fr_FR -f UTF-8 fr_FR.UTF-8
(lfs chroot) root:/sources/glibc-2.32/build# localedef -i it_IT -f ISO-8859-1 it_IT
(lfs chroot) root:/sources/glibc-2.32/build# localedef -i it_IT -f UTF-8 it_IT.UTF-8
(lfs chroot) root:/sources/glibc-2.32/build# localedef -i ja_JP -f EUC-JP ja_JP
(lfs chroot) root:/sources/glibc-2.32/build# localedef -i ja_JP -f SHIFT_JIS ja_JP.SIJS 2> /dev/null || true
(lfs chroot) root:/sources/glibc-2.32/build# localedef -i ja_JP -f UTF-8 ja_JP.UTF-8
(lfs chroot) root:/sources/glibc-2.32/build# localedef -i ru_RU -f KOI8-R ru_RU.KOI8-R
(lfs chroot) root:/sources/glibc-2.32/build# localedef -i ru_RU -f UTF-8 ru_RU.UTF-8
(lfs chroot) root:/sources/glibc-2.32/build# localedef -i tr_TR -f UTF-8 tr_TR.UTF-8
(lfs chroot) root:/sources/glibc-2.32/build# localedef -i zh_CN -f GB18030 zh_CN.GB18030
(lfs chroot) root:/sources/glibc-2.32/build# localedef -i zh_HK -f BIG5-HKSCS zh_HK.BIG5-HKSCS
```

此外，为您自己的国家、语言和字符集安装语言环境。

或者，使用以下耗时的命令一次安装 glibc-2.32/localedata/SUPPORTED 文件中列出的所有语言环境（它包括上面列出的每个语言环境以及更多）：

```sh
# 以下命令大约耗时20分钟
(lfs chroot) root:/sources/glibc-2.32/build# time { make localedata/install-locales; }

# 运行结束后输出以下信息：
make[2]: Leaving directory '/sources/glibc-2.32/localedata'
make[1]: Leaving directory '/sources/glibc-2.32'

real    17m0.232s
user    15m54.952s
sys     0m54.196s
```

然后使用 localedef 命令创建和安装 glibc-2.32/localedata/SUPPORTED 文件中未列出的语言环境，以防万一您需要它们。

注意：Glibc 现在在解析国际化域名时使用 libidn2。这是一个运行时依赖。如果需要此功能，安装 libidn2 的说明位于 BLFS libidn2 页面中。

#### 8.8.2 配置 Glibc

#### 8.8.2.1 添加 nsswitch.conf

需要创建 /etc/nsswitch.conf 文件，因为 Glibc 默认设置是不能在网络环境中工作。

通过运行以下命令创建一个新文件 /etc/nsswitch.conf：

```sh
cat > /etc/nsswitch.conf << "EOF"
# Begin /etc/nsswitch.conf

passwd: files
group: files
shadow: files

hosts: files dns
networks: files

protocols: files
services: files
ethers: files
rpc: files

# End /etc/nsswitch.conf
EOF
```

#### 8.8.2.2 添加时区数据

使用以下命令安装和设置时区数据：

```sh
# 解压软件包
(lfs chroot) root:/sources/glibc-2.32/build# tar -xf ../../tzdata2020a.tar.gz

# 设置环境变量并创建目录
(lfs chroot) root:/sources/glibc-2.32/build# ZONEINFO=/usr/share/zoneinfo
(lfs chroot) root:/sources/glibc-2.32/build# mkdir -pv $ZONEINFO/{posix,right}
mkdir: created directory '/usr/share/zoneinfo/posix'
mkdir: created directory '/usr/share/zoneinfo/right'

# 执行以下脚本
for tz in etcetera southamerica northamerica europe africa antarctica  \
          asia australasia backward pacificnew systemv; do
    zic -L /dev/null   -d $ZONEINFO       ${tz}
    zic -L /dev/null   -d $ZONEINFO/posix ${tz}
    zic -L leapseconds -d $ZONEINFO/right ${tz}
done

##### 脚本输出信息：
warning: "leapseconds", line 79: "#expires" is obsolescent; use "Expires"
warning: "leapseconds", line 79: "#expires" is obsolescent; use "Expires"
warning: "leapseconds", line 79: "#expires" is obsolescent; use "Expires"
warning: "leapseconds", line 79: "#expires" is obsolescent; use "Expires"
warning: "leapseconds", line 79: "#expires" is obsolescent; use "Expires"
warning: "leapseconds", line 79: "#expires" is obsolescent; use "Expires"
warning: "leapseconds", line 79: "#expires" is obsolescent; use "Expires"
warning: "leapseconds", line 79: "#expires" is obsolescent; use "Expires"
warning: "leapseconds", line 79: "#expires" is obsolescent; use "Expires"
warning: "leapseconds", line 79: "#expires" is obsolescent; use "Expires"
warning: "leapseconds", line 79: "#expires" is obsolescent; use "Expires"

# 继续最后的操作
(lfs chroot) root:/sources/glibc-2.32/build# cp -v zone.tab zone1970.tab iso3166.tab $ZONEINFO
'zone.tab' -> '/usr/share/zoneinfo/zone.tab'
'zone1970.tab' -> '/usr/share/zoneinfo/zone1970.tab'
'iso3166.tab' -> '/usr/share/zoneinfo/iso3166.tab'
(lfs chroot) root:/sources/glibc-2.32/build# zic -d $ZONEINFO -p America/New_York
(lfs chroot) root:/sources/glibc-2.32/build# unset ZONEINFO
```

确定本地时区的一种方法是运行以下脚本：

```sh
(lfs chroot) root:/sources/glibc-2.32/build# tzselect

# 脚本运行后，依次回答"所在地区"、"所在国家"、"地区时间"三项问题，随后确定即可，即需要依次输入"4"、"9"、"1"、"1"
# 完整例程如下：
(lfs chroot) root:/sources/glibc-2.32/build# tzselect
Please identify a location so that time zone rules can be set correctly.
Please select a continent, ocean, "coord", or "TZ".
 1) Africa
 2) Americas
 3) Antarctica
 4) Asia
 5) Atlantic Ocean
 6) Australia
 7) Europe
 8) Indian Ocean
 9) Pacific Ocean
10) coord - I want to use geographical coordinates.
11) TZ - I want to specify the timezone using the Posix TZ format.
#? 4
Please select a country whose clocks agree with yours.
 1) Afghanistan           18) Israel                35) Palestine
 2) Armenia               19) Japan                 36) Philippines
 3) Azerbaijan            20) Jordan                37) Qatar
 4) Bahrain               21) Kazakhstan            38) Russia
 5) Bangladesh            22) Korea (North)         39) Saudi Arabia
 6) Bhutan                23) Korea (South)         40) Singapore
 7) Brunei                24) Kuwait                41) Sri Lanka
 8) Cambodia              25) Kyrgyzstan            42) Syria
 9) China                 26) Laos                  43) Taiwan
10) Cyprus                27) Lebanon               44) Tajikistan
11) East Timor            28) Macau                 45) Thailand
12) Georgia               29) Malaysia              46) Turkmenistan
13) Hong Kong             30) Mongolia              47) United Arab Emirates
14) India                 31) Myanmar (Burma)       48) Uzbekistan
15) Indonesia             32) Nepal                 49) Vietnam
16) Iran                  33) Oman                  50) Yemen
17) Iraq                  34) Pakistan
#? 9
Please select one of the following timezones.
1) Beijing Time
2) Xinjiang Time
#? 1

The following information has been given:

        China
        Beijing Time

Therefore TZ='Asia/Shanghai' will be used.
Selected time is now:   Wed Dec 22 09:15:01 CST 2021.
Universal Time is now:  Wed Dec 22 01:15:01 UTC 2021.
Is the above information OK?
1) Yes
2) No
#? 1

You can make this change permanent for yourself by appending the line
        TZ='Asia/Shanghai'; export TZ
to the file '.profile' in your home directory; then log out and log in again.

Here is that TZ value again, this time on standard output so that you
can use the /usr/bin/tzselect command in shell scripts:
Asia/Shanghai
(lfs chroot) root:/sources/glibc-2.32/build# 
```

回答几个关于位置的问题后，脚本将输出时区的名称（例如，美国/埃德蒙顿）。在 /usr/share/zoneinfo 中还列出了一些其他可能的时区，例如 Canada/Eastern 或 EST5EDT，它们未被脚本识别但可以使用。

在中国，您得到的时区应该是 `Asia/Shanghai`

然后通过运行创建 /etc/localtime 文件：

ln -sfv /usr/share/zoneinfo/<xxx> /etc/localtime (不要直接运行此命令)

注意：将 `<xxx>` 替换为所选时区的名称（例如，`Canada/Eastern`）。

请运行以下命令：

```sh
(lfs chroot) root:/sources/glibc-2.32/build# ln -sfv /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
'/etc/localtime' -> '/usr/share/zoneinfo/Asia/Shanghai'
```

##### 8.8.2.3 配置动态加载器

默认情况下，动态加载器 (/lib/ld-linux.so.2) 在 /lib 和 /usr/lib 中搜索程序运行时所需的动态库。但是，如果目录中有除 `/lib` 和 `/usr/lib` 之外的库，则需要将这些库添加到 /etc/ld.so.conf 文件中，以便动态加载程序找到它们。通常已知的包含附加库的两个目录是 `/usr/local/lib` 和 `/opt/lib`，因此将这些目录添加到动态加载程序的搜索路径中。

通过运行以下命令创建一个新文件 /etc/ld.so.conf：

```sh
cat > /etc/ld.so.conf << "EOF"
# Begin /etc/ld.so.conf
/usr/local/lib
/opt/lib

EOF
```

如果需要，动态加载器还可以搜索目录并包含在那里找到的文件的内容。通常这个包含目录中的文件是一行指定所需的库路径。要添加此功能，请运行以下命令：

```sh
cat >> /etc/ld.so.conf << "EOF"
# Add an include directory
include /etc/ld.so.conf.d/*.conf

EOF
mkdir -pv /etc/ld.so.conf.d

# 此段代码例程如下（不要重复执行）：
(lfs chroot) root:/sources/glibc-2.32/build# cat >> /etc/ld.so.conf << "EOF"
> # Add an include directory
> include /etc/ld.so.conf.d/*.conf
> 
> EOF
(lfs chroot) root:/sources/glibc-2.32/build# mkdir -pv /etc/ld.so.conf.d
mkdir: created directory '/etc/ld.so.conf.d'
```

清理软件包

```sh
(lfs chroot) root:/sources/glibc-2.32/build# cd ../..
(lfs chroot) root:/sources# rm -rf glibc-2.32
```

### 8.9 安装 Zlib-1.2.11

Zlib 包含一些程序使用的压缩和解压缩例程

解压软件包

```sh
(lfs chroot) root:/sources# tar xf zlib-1.2.11.tar.xz 
(lfs chroot) root:/sources# cd zlib-1.2.11
```

准备配置并编译 (大约耗时10秒)

```sh
time { ./configure --prefix=/usr && make; }

# 编译完成后输出以下信息：
gcc -O3 -D_LARGEFILE64_SOURCE=1 -DHAVE_HIDDEN -o example64 example64.o -L. libz.a
gcc -O3 -D_LARGEFILE64_SOURCE=1 -DHAVE_HIDDEN -I. -D_FILE_OFFSET_BITS=64 -c -o minigzip64.o test/minigzip.c
gcc -O3 -D_LARGEFILE64_SOURCE=1 -DHAVE_HIDDEN -o minigzip64 minigzip64.o -L. libz.a

real    0m9.545s
user    0m8.401s
sys     0m0.888s
```

若要测试结果，请执行以下指令 (大约耗时1秒)：

```sh
time { make check; }

# 测试完成后输出以下信息：
after inflateSync(): hello, hello!
inflate with dictionary: hello, hello!
                *** zlib 64-bit test OK ***

real    0m0.064s
user    0m0.020s
sys     0m0.029s
```

安装软件包 (大约耗时1秒)：

```sh
time { make install; }

# 安装完成后输出以下信息：
cp zlib.h zconf.h /usr/include
chmod 644 /usr/include/zlib.h /usr/include/zconf.h

real    0m0.085s
user    0m0.038s
sys     0m0.025s
```

共享库需要移动到 /lib，因此需要重新创建 /usr/lib 中的 .so 文件：

```sh
(lfs chroot) root:/sources/zlib-1.2.11# mv -v /usr/lib/libz.so.* /lib
renamed '/usr/lib/libz.so.1' -> '/lib/libz.so.1'
renamed '/usr/lib/libz.so.1.2.11' -> '/lib/libz.so.1.2.11'
(lfs chroot) root:/sources/zlib-1.2.11# ln -sfv ../../lib/$(readlink /usr/lib/libz.so) /usr/lib/libz.so
'/usr/lib/libz.so' -> '../../lib/libz.so.1.2.11'
```

清除软件包

```sh
(lfs chroot) root:/sources/zlib-1.2.11# cd ..
(lfs chroot) root:/sources# rm -rf zlib-1.2.11
```

### 8.10 安装 Bzip2-1.0.8

Bzip2 包包含用于压缩和解压缩文件的程序。使用 bzip2 压缩文本文件比使用传统 gzip 产生更好的压缩百分比。

解压软件包

```sh
(lfs chroot) root:/sources# tar xf bzip2-1.0.8.tar.gz 
(lfs chroot) root:/sources# cd bzip2-1.0.8
```

应用一个补丁来安装这个包的文档：

```sh
(lfs chroot) root:/sources/bzip2-1.0.8# patch -Np1 -i ../bzip2-1.0.8-install_docs-1.patch
patching file Makefile
```

以下命令确保符号链接的安装是相对的：

```sh
(lfs chroot) root:/sources/bzip2-1.0.8# sed -i 's@\(ln -s -f \)$(PREFIX)/bin/@\1@' Makefile
```

确保手册页安装到正确的位置：

```sh
(lfs chroot) root:/sources/bzip2-1.0.8# sed -i "s@(PREFIX)/man@(PREFIX)/share/man@g" Makefile
```

准备配置

```sh
(lfs chroot) root:/sources/bzip2-1.0.8# make -f Makefile-libbz2_so
(lfs chroot) root:/sources/bzip2-1.0.8# make clean

# 完成后输出以下信息：
rm -f *.o libbz2.a bzip2 bzip2recover \
sample1.rb2 sample2.rb2 sample3.rb2 \
sample1.tst sample2.tst sample3.tst
(lfs chroot) root:/sources/bzip2-1.0.8# 
```

编译并安装 (大约耗时5秒)

```sh
(lfs chroot) root:/sources/bzip2-1.0.8# time { make && make PREFIX=/usr install; }

# 编译安装完成后输出信息：
cp -f manual.ps /usr/share/doc/bzip2-1.0.8
cp -f bzip2.txt /usr/share/doc/bzip2-1.0.8
chmod a+r /usr/share/doc/bzip2-1.0.8/manual.html
chmod a+r /usr/share/doc/bzip2-1.0.8/manual.pdf
chmod a+r /usr/share/doc/bzip2-1.0.8/manual.ps
chmod a+r /usr/share/doc/bzip2-1.0.8/bzip2.txt

real    0m6.787s
user    0m6.069s
sys     0m0.543s
```

将共享的 bzip2 二进制文件安装到 /bin 目录中，制作一些必要的符号链接，并进行清理：

```sh
(lfs chroot) root:/sources/bzip2-1.0.8# cp -v bzip2-shared /bin/bzip2
'bzip2-shared' -> '/bin/bzip2'
(lfs chroot) root:/sources/bzip2-1.0.8# cp -av libbz2.so* /lib
'libbz2.so.1.0' -> '/lib/libbz2.so.1.0'
'libbz2.so.1.0.8' -> '/lib/libbz2.so.1.0.8'
(lfs chroot) root:/sources/bzip2-1.0.8# ln -sv ../../lib/libbz2.so.1.0 /usr/lib/libbz2.so
'/usr/lib/libbz2.so' -> '../../lib/libbz2.so.1.0'
(lfs chroot) root:/sources/bzip2-1.0.8# rm -v /usr/bin/{bunzip2,bzcat,bzip2}
removed '/usr/bin/bunzip2'
removed '/usr/bin/bzcat'
removed '/usr/bin/bzip2'
(lfs chroot) root:/sources/bzip2-1.0.8# ln -sv bzip2 /bin/bunzip2
'/bin/bunzip2' -> 'bzip2'
(lfs chroot) root:/sources/bzip2-1.0.8# ln -sv bzip2 /bin/bzcat
'/bin/bzcat' -> 'bzip2'
```

清理安装包

```sh
(lfs chroot) root:/sources/bzip2-1.0.8# cd ..
(lfs chroot) root:/sources# rm -rf bzip2-1.0.8
```

### 8.11 安装 Xz-5.2.5

Xz 包含用于压缩和解压缩文件的程序。它为 lzma 和较新的 xz 压缩格式提供了功能。使用 xz 压缩文本文件比使用传统的 gzip 或 bzip2 命令产生更好的压缩百分比。

解压软件包

```sh
(lfs chroot) root:/sources# tar xf xz-5.2.5.tar.xz 
(lfs chroot) root:/sources# cd xz-5.2.5
```

配置并编译检查 (大约耗时1分钟)

```sh
time { ./configure --prefix=/usr  \
            --disable-static      \
            --docdir=/usr/share/doc/xz-5.2.5 && make && make check; }

# 检查完成后输出信息：
==================
All 9 tests passed
==================
make[2]: Leaving directory '/sources/xz-5.2.5/tests'
make[1]: Leaving directory '/sources/xz-5.2.5/tests'
make[1]: Entering directory '/sources/xz-5.2.5'
make[1]: Leaving directory '/sources/xz-5.2.5'

real    0m57.326s
user    0m41.632s
sys     0m12.493s
```

安装软件包 (大约耗时2秒)：

```sh
(lfs chroot) root:/sources/xz-5.2.5# time { make install; }

# 安装完成后的输出：
make[2]: Leaving directory '/sources/xz-5.2.5'
make[1]: Leaving directory '/sources/xz-5.2.5'

real    0m2.158s
user    0m1.387s
sys     0m0.602s
```

确保所有基本文件都在正确的目录中：

```sh
(lfs chroot) root:/sources/xz-5.2.5# mv -v /usr/bin/{lzma,unlzma,lzcat,xz,unxz,xzcat} /bin
renamed '/usr/bin/lzma' -> '/bin/lzma'
renamed '/usr/bin/unlzma' -> '/bin/unlzma'
renamed '/usr/bin/lzcat' -> '/bin/lzcat'
renamed '/usr/bin/xz' -> '/bin/xz'
renamed '/usr/bin/unxz' -> '/bin/unxz'
renamed '/usr/bin/xzcat' -> '/bin/xzcat'
(lfs chroot) root:/sources/xz-5.2.5# mv -v /usr/lib/liblzma.so.* /lib
renamed '/usr/lib/liblzma.so.5' -> '/lib/liblzma.so.5'
renamed '/usr/lib/liblzma.so.5.2.5' -> '/lib/liblzma.so.5.2.5'
(lfs chroot) root:/sources/xz-5.2.5# ln -svf ../../lib/$(readlink /usr/lib/liblzma.so) /usr/lib/liblzma.so
'/usr/lib/liblzma.so' -> '../../lib/liblzma.so.5.2.5'
```

清理软件包

```sh
(lfs chroot) root:/sources/xz-5.2.5# cd ..
(lfs chroot) root:/sources# rm -rf xz-5.2.5
```

### 8.12 安装 Zstd-1.4.5

Zstandard 是一种实时压缩算法，提供高压缩率。它提供了非常广泛的压缩/速度权衡，同时由非常快速的解码器提供支持。

解压软件包

```sh
(lfs chroot) root:/sources# tar xf zstd-1.4.5.tar.gz 
(lfs chroot) root:/sources# cd zstd-1.4.5
```

编译并安装 (大约耗时2分钟)

```sh
(lfs chroot) root:/sources/zstd-1.4.5# time { make && make prefix=/usr install; }

# 安装完成后输出信息：
creating versioned links
Installing shared library
Installing includes
zstd static and shared library installed
make[1]: Leaving directory '/sources/zstd-1.4.5/lib'
make[1]: Entering directory '/sources/zstd-1.4.5/programs'
Installing binaries
Installing man pages
zstd installation completed
make[1]: Leaving directory '/sources/zstd-1.4.5/programs'

real    2m37.485s
user    2m31.136s
sys     0m4.358s
```

删除静态库并将共享库移动到 /lib。此外，/usr/lib 中的 .so 文件将需要重新创建：

```sh
(lfs chroot) root:/sources/zstd-1.4.5# rm -v /usr/lib/libzstd.a
removed '/usr/lib/libzstd.a'
(lfs chroot) root:/sources/zstd-1.4.5# mv -v /usr/lib/libzstd.so.* /lib
renamed '/usr/lib/libzstd.so.1' -> '/lib/libzstd.so.1'
renamed '/usr/lib/libzstd.so.1.4.5' -> '/lib/libzstd.so.1.4.5'
(lfs chroot) root:/sources/zstd-1.4.5# ln -sfv ../../lib/$(readlink /usr/lib/libzstd.so) /usr/lib/libzstd.so
'/usr/lib/libzstd.so' -> '../../lib/libzstd.so.1.4.5'
```

清除软件包

```sh
(lfs chroot) root:/sources/zstd-1.4.5# cd ..
(lfs chroot) root:/sources# rm -rf zstd-1.4.5
```

### 8.13 安装 File-5.39

File 包含一个用于确定给定文件或文件类型的实用程序。

解压软件包

```sh
(lfs chroot) root:/sources# tar xf file-5.39.tar.gz 
(lfs chroot) root:/sources# cd file-5.39
```

配置编译并检查 (大约耗时30秒)

```sh
time { ./configure --prefix=/usr && make && make check; }

# 检查完后的输出信息：
make[2]: Leaving directory '/sources/file-5.39/tests'
make[1]: Leaving directory '/sources/file-5.39/tests'
Making check in doc
make[1]: Entering directory '/sources/file-5.39/doc'
make[1]: Nothing to be done for 'check'.
make[1]: Leaving directory '/sources/file-5.39/doc'
Making check in python
make[1]: Entering directory '/sources/file-5.39/python'
make[1]: Nothing to be done for 'check'.
make[1]: Leaving directory '/sources/file-5.39/python'
make[1]: Entering directory '/sources/file-5.39'
make[1]: Leaving directory '/sources/file-5.39'

real    0m28.106s
user    0m19.706s
sys     0m6.196s
```

安装软件包

```sh
(lfs chroot) root:/sources/file-5.39# time { make install; }

# 安装后的输出信息：
make[2]: Nothing to be done for 'install-data-am'.
make[2]: Leaving directory '/sources/file-5.39'
make[1]: Leaving directory '/sources/file-5.39'

real    0m0.566s
user    0m0.383s
sys     0m0.140s
```

清除软件包

```sh
(lfs chroot) root:/sources/file-5.39# cd ..
(lfs chroot) root:/sources# rm -rf file-5.39
```

### 8.14 安装 Readline-8.0

Readline 包是一组提供命令行编辑和历史记录功能的库。

安装软件包

```sh
(lfs chroot) root:/sources# tar xf readline-8.0.tar.gz 
(lfs chroot) root:/sources# cd readline-8.0
```

重新安装 Readline 将导致旧库移动到 <libraryname>.old。虽然这通常不是问题，但在某些情况下，它可能会触发 ldconfig 中的链接错误。这可以通过发出以下两个 sed 来避免：

```sh
(lfs chroot) root:/sources/readline-8.0# sed -i '/MV.*old/d' Makefile.in
(lfs chroot) root:/sources/readline-8.0# sed -i '/{OLDSUFF}/c:' support/shlib-install
```

配置并编译安装 (大约耗时20秒)

```sh
time { ./configure --prefix=/usr   \
            --disable-static       \
            --with-curses          \
            --docdir=/usr/share/doc/readline-8.0 && make SHLIB_LIBS="-lncursesw" && make SHLIB_LIBS="-lncursesw" install; }

# 安装完成后输出信息：
make[1]: Entering directory '/sources/readline-8.0/shlib'
/bin/sh ../support/mkdirs /usr/lib
/bin/sh ../support/mkdirs /usr/bin
/bin/sh ../support/shlib-install -O linux-gnu -V pc -d /usr/lib -b /usr/bin -i "/usr/bin/install -c -m 644" libhistory.so.8.0
/bin/sh ../support/shlib-install -O linux-gnu -V pc -d /usr/lib -b /usr/bin -i "/usr/bin/install -c -m 644" libreadline.so.8.0
install: you may need to run ldconfig
make[1]: Leaving directory '/sources/readline-8.0/shlib'

real    0m20.950s
user    0m14.657s
sys     0m4.554s
```

现在将动态库移动到更合适的位置并修复一些权限和符号链接：

```sh
(lfs chroot) root:/sources/readline-8.0# mv -v /usr/lib/lib{readline,history}.so.* /lib
renamed '/usr/lib/libreadline.so.8' -> '/lib/libreadline.so.8'
renamed '/usr/lib/libreadline.so.8.0' -> '/lib/libreadline.so.8.0'
renamed '/usr/lib/libhistory.so.8' -> '/lib/libhistory.so.8'
renamed '/usr/lib/libhistory.so.8.0' -> '/lib/libhistory.so.8.0'
(lfs chroot) root:/sources/readline-8.0# chmod -v u+w /lib/lib{readline,history}.so.*
mode of '/lib/libreadline.so.8' retained as 0755 (rwxr-xr-x)
mode of '/lib/libreadline.so.8.0' retained as 0755 (rwxr-xr-x)
mode of '/lib/libhistory.so.8' retained as 0755 (rwxr-xr-x)
mode of '/lib/libhistory.so.8.0' retained as 0755 (rwxr-xr-x)
(lfs chroot) root:/sources/readline-8.0# ln -sfv ../../lib/$(readlink /usr/lib/libreadline.so) /usr/lib/libreadline.so
'/usr/lib/libreadline.so' -> '../../lib/libreadline.so.8'
(lfs chroot) root:/sources/readline-8.0# ln -sfv ../../lib/$(readlink /usr/lib/libhistory.so ) /usr/lib/libhistory.so
'/usr/lib/libhistory.so' -> '../../lib/libhistory.so.8'
```

如果需要，请安装文档：

```sh
(lfs chroot) root:/sources/readline-8.0# install -v -m644 doc/*.{ps,pdf,html,dvi} /usr/share/doc/readline-8.0
'doc/history.ps' -> '/usr/share/doc/readline-8.0/history.ps'
'doc/history_3.ps' -> '/usr/share/doc/readline-8.0/history_3.ps'
'doc/readline.ps' -> '/usr/share/doc/readline-8.0/readline.ps'
'doc/readline_3.ps' -> '/usr/share/doc/readline-8.0/readline_3.ps'
'doc/rluserman.ps' -> '/usr/share/doc/readline-8.0/rluserman.ps'
'doc/history.pdf' -> '/usr/share/doc/readline-8.0/history.pdf'
'doc/readline.pdf' -> '/usr/share/doc/readline-8.0/readline.pdf'
'doc/rluserman.pdf' -> '/usr/share/doc/readline-8.0/rluserman.pdf'
'doc/history.html' -> '/usr/share/doc/readline-8.0/history.html'
'doc/readline.html' -> '/usr/share/doc/readline-8.0/readline.html'
'doc/rluserman.html' -> '/usr/share/doc/readline-8.0/rluserman.html'
'doc/history.dvi' -> '/usr/share/doc/readline-8.0/history.dvi'
'doc/readline.dvi' -> '/usr/share/doc/readline-8.0/readline.dvi'
'doc/rluserman.dvi' -> '/usr/share/doc/readline-8.0/rluserman.dvi'
```

清除安装包

```sh
(lfs chroot) root:/sources/readline-8.0# cd ..
(lfs chroot) root:/sources# rm -rf readline-8.0
```

### 8.15 安装 M4-1.4.18

M4 封装包含一个宏处理器。

解压软件包

```sh
(lfs chroot) root:/sources# tar xf m4-1.4.18.tar.xz 
(lfs chroot) root:/sources# cd m4-1.4.18
```

首先执行 glibc-2.28 及更高版本所需的一些修复：

```sh
(lfs chroot) root:/sources/m4-1.4.18# sed -i 's/IO_ftrylockfile/IO_EOF_SEEN/' lib/*.c
(lfs chroot) root:/sources/m4-1.4.18# echo "#define _IO_IN_BACKUP 0x100" >> lib/stdio-impl.h
```

配置编译并检查

```sh
(lfs chroot) root:/sources/m4-1.4.18# time { ./configure --prefix=/usr && make && make check; }

# 检查完成后输出信息：
============================================================================
Testsuite summary for GNU M4 1.4.18
============================================================================
# TOTAL: 170
# PASS:  157
# SKIP:  13
# XFAIL: 0
# FAIL:  0
# XPASS: 0
# ERROR: 0
============================================================================
make[6]: Leaving directory '/sources/m4-1.4.18/tests'
make[5]: Leaving directory '/sources/m4-1.4.18/tests'
make[4]: Leaving directory '/sources/m4-1.4.18/tests'
make[3]: Leaving directory '/sources/m4-1.4.18/tests'
make[2]: Leaving directory '/sources/m4-1.4.18/tests'
make[1]: Leaving directory '/sources/m4-1.4.18'

real    1m51.158s
user    1m10.888s
sys     0m22.973s
```

安装软件包

```sh
(lfs chroot) root:/sources/m4-1.4.18# time { make install; }

# 安装完成后输出信息：
make[4]: Entering directory '/sources/m4-1.4.18/tests'
make[5]: Entering directory '/sources/m4-1.4.18/tests'
make[5]: Nothing to be done for 'install-exec-am'.
make[5]: Nothing to be done for 'install-data-am'.
make[5]: Leaving directory '/sources/m4-1.4.18/tests'
make[4]: Leaving directory '/sources/m4-1.4.18/tests'
make[3]: Leaving directory '/sources/m4-1.4.18/tests'
make[2]: Leaving directory '/sources/m4-1.4.18/tests'
make[1]: Leaving directory '/sources/m4-1.4.18'

real    0m1.232s
user    0m0.868s
sys     0m0.193s
```

清理软件包

```sh
(lfs chroot) root:/sources/m4-1.4.18# cd ..
(lfs chroot) root:/sources# rm -rf m4-1.4.18
```

### 8.16 安装 Bc-3.1.5

Bc 包含一种任意精度的数字处理语言。

解压软件包

```sh
(lfs chroot) root:/sources# tar xf bc-3.1.5.tar.xz 
(lfs chroot) root:/sources# cd bc-3.1.5
```

配置编译并检查

```sh
time { PREFIX=/usr CC=gcc CFLAGS="-std=c99" ./configure.sh -G -O3 && make && make test; }

# 检查完成后输出信息：
Running dc binary file test...pass
Running dc binary stdin test...pass

All dc tests passed.

***********************************************************************

real    0m12.970s
user    0m10.569s
sys     0m2.186s
```

安装软件包

```sh
(lfs chroot) root:/sources/bc-3.1.5# time { make install; }

# 安装完成后输出信息：
./locale_install.sh /usr/share/locale/%L/%N bc 
./safe-install.sh -Dm644 manuals/bc.1 /usr/share/man/man1/bc.1
./safe-install.sh -Dm644 manuals/dc.1 /usr/share/man/man1/dc.1
./install.sh /usr/bin ""

real    0m22.463s
user    0m20.792s
sys     0m1.293s
```

清理软件包

```sh
(lfs chroot) root:/sources/bc-3.1.5# cd ..
(lfs chroot) root:/sources# rm -rf bc-3.1.5
```

### 8.17 安装 Flex-2.6.4

Flex 包含一个实用程序，用于生成识别文本模式的程序。

解压软件包

```sh
(lfs chroot) root:/sources# tar xf flex-2.6.4.tar.gz 
(lfs chroot) root:/sources# cd flex-2.6.4
```

配置编译并检查

```sh
time { ./configure --prefix=/usr --docdir=/usr/share/doc/flex-2.6.4 && make && make check; }

# 检查完成后输出信息：
============================================================================
Testsuite summary for the fast lexical analyser generator 2.6.4
============================================================================
# TOTAL: 114
# PASS:  114
# SKIP:  0
# XFAIL: 0
# FAIL:  0
# XPASS: 0
# ERROR: 0
============================================================================
make[3]: Leaving directory '/sources/flex-2.6.4/tests'
make[2]: Leaving directory '/sources/flex-2.6.4/tests'
make[1]: Leaving directory '/sources/flex-2.6.4/tests'
Making check in tools
make[1]: Entering directory '/sources/flex-2.6.4/tools'
make[1]: Nothing to be done for 'check'.
make[1]: Leaving directory '/sources/flex-2.6.4/tools'
make[1]: Entering directory '/sources/flex-2.6.4'
make[1]: Nothing to be done for 'check-am'.
make[1]: Leaving directory '/sources/flex-2.6.4'

real    1m46.022s
user    1m29.591s
sys     0m13.206s
```

安装软件包

```sh
(lfs chroot) root:/sources/flex-2.6.4# time { make install; }

# 安装完成后输出信息：
make[2]: Leaving directory '/sources/flex-2.6.4'
make[1]: Leaving directory '/sources/flex-2.6.4'

real    0m0.916s
user    0m0.586s
sys     0m0.222s
```

一些程序还不了解 flex 并尝试运行它的前身 lex。要支持这些程序，请创建一个名为 lex 的符号链接，该链接以 lex 仿真模式运行 flex：

```sh
(lfs chroot) root:/sources/flex-2.6.4# ln -sv flex /usr/bin/lex
'/usr/bin/lex' -> 'flex'
```

清理软件包

```sh
(lfs chroot) root:/sources/flex-2.6.4# cd ..
(lfs chroot) root:/sources# rm -rf flex-2.6.4
```

### 8.18 安装 Binutils-2.35

Binutils 包含一个链接器、一个汇编器和其他用于处理目标文件的工具。

解压软件包

```sh
(lfs chroot) root:/sources# tar xf binutils-2.35.tar.xz 
(lfs chroot) root:/sources# cd binutils-2.35
```

通过执行一个简单的测试来验证 PTY 在 chroot 环境中是否正常工作：

```sh
(lfs chroot) root:/sources/binutils-2.35# expect -c "spawn ls"
spawn ls
```

此命令应输出以下内容：

```sh
spawn ls
```

相反，如果输出包含以下消息，则环境未设置为正确的 PTY 操作。在运行 Binutils 和 GCC 的测试套件之前，需要解决这个问题：

```sh
The system has no more ptys.
Ask your system administrator to create more.
```

现在删除一个阻止测试运行到完成的测试：

```sh
(lfs chroot) root:/sources/binutils-2.35# sed -i '/@\tincremental_copy/d' gold/testsuite/Makefile.in
```

Binutils 文档建议在专用构建目录中构建 Binutils

```sh
(lfs chroot) root:/sources/binutils-2.35# mkdir -v build
mkdir: created directory 'build'
(lfs chroot) root:/sources/binutils-2.35# cd build/
```

配置编译并检查，本节中的 Binutils 测试套件被认为是至关重要的。在任何情况下都不要跳过它。

```sh
time { ../configure --prefix=/usr   \
             --enable-gold          \
             --enable-ld=default    \
             --enable-plugins       \
             --enable-shared        \
             --disable-werror       \
             --enable-64-bit-bfd    \
             --with-system-zlib && make tooldir=/usr && make -k check; }

# 检查完成后输出信息：
make[3]: Leaving directory '/sources/binutils-2.35/build/libiberty/testsuite'
make[2]: Leaving directory '/sources/binutils-2.35/build/libiberty'
make[1]: Nothing to be done for 'check-target'.
make[1]: Leaving directory '/sources/binutils-2.35/build'

real    28m33.890s
user    22m51.026s
sys     5m18.164s
```

安装软件包

```sh
time { make tooldir=/usr install; }

# 安装完成后输出信息：
make[3]: Leaving directory '/sources/binutils-2.35/build/libctf'
make[2]: Leaving directory '/sources/binutils-2.35/build/libctf'
make[1]: Nothing to be done for 'install-target'.
make[1]: Leaving directory '/sources/binutils-2.35/build'

real    0m9.725s
user    0m5.144s
sys     0m2.868s
```

清理软件包

```sh
(lfs chroot) root:/sources/binutils-2.35/build# cd ../..
(lfs chroot) root:/sources# rm -rf binutils-2.35
```

### 8.19 安装 GMP-6.2.0

GMP 包含数学库。这些具有用于任意精度算术的有用函数。

解压软件包

```sh
(lfs chroot) root:/sources# tar xf gmp-6.2.0.tar.xz 
(lfs chroot) root:/sources# cd gmp-6.2.0
```

注意：如果您正在为 32 位 x86 构建，但您有一个能够运行 64 位代码的 CPU，并且您在环境中指定了 CFLAGS，配置脚本将尝试配置 64 位并失败。通过调用下面的配置命令来避免这种情况 `ABI=32 ./configure ...`(我不运行此代码)。

GMP 的默认设置生成针对主机处理器优化的库。如果需要适用于能力低于宿主机 CPU 的处理器的库，可以通过运行以下命令来创建通用库：

```sh
cp -v configfsf.guess config.guess  (我不运行此代码)
cp -v configfsf.sub   config.sub    (我不运行此代码)
```

配置编译并生成 HTML 文档

```sh
time { ./configure --prefix=/usr   \
            --enable-cxx           \
            --disable-static       \
            --docdir=/usr/share/doc/gmp-6.2.0 && make && make html; }

# 编译完成后输出信息：
make[1]: Leaving directory '/sources/gmp-6.2.0/doc'
make[1]: Entering directory '/sources/gmp-6.2.0'
make[1]: Nothing to be done for 'html-am'.
make[1]: Leaving directory '/sources/gmp-6.2.0'

real    2m16.889s
user    1m46.007s
sys     0m24.619s
```

！重要：本节中的 GMP 测试套件被认为是至关重要的。在任何情况下都不要跳过它。

```sh
time { make check 2>&1 | tee gmp-check-log; }

# 测试完成后输出信息：
make[2]: Leaving directory '/sources/gmp-6.2.0/doc'
make[2]: Entering directory '/sources/gmp-6.2.0'
make[2]: Leaving directory '/sources/gmp-6.2.0'
make[1]: Leaving directory '/sources/gmp-6.2.0'

real    2m18.843s
user    2m0.139s
sys     0m14.226s
```

注意：gmp 中的代码针对构建它的处理器进行了高度优化。有时，检测处理器的代码会错误地识别系统功能，并且在使用 gmp 库的测试或其他应用程序中会出现错误并显示“非法指令”(Illegal instruction)消息。在这种情况下，应该使用选项 `--build=x86_64-unknown-linux-gnu` 重新配置 gmp 并重新构建。

确保测试套件中的所有 197 个测试都通过。通过发出以下命令检查结果：

```sh
time { awk '/# PASS:/{total+=$3} ; END{print total}' gmp-check-log; }

# 测试完成后输出信息：
(lfs chroot) root:/sources/gmp-6.2.0# time { awk '/# PASS:/{total+=$3} ; END{print total}' gmp-check-log; }
197

real    0m0.003s
user    0m0.002s
sys     0m0.000s
```

安装软件包及其文档：

```sh
(lfs chroot) root:/sources/gmp-6.2.0# time { make install && make install-html; }

# 安装完成后输出信息：
make[1]: Leaving directory '/sources/gmp-6.2.0/doc'
make[1]: Entering directory '/sources/gmp-6.2.0'
make[1]: Nothing to be done for 'install-html-am'.
make[1]: Leaving directory '/sources/gmp-6.2.0'

real    0m1.319s
user    0m0.813s
sys     0m0.331s
```

清理软件包

```sh
(lfs chroot) root:/sources/gmp-6.2.0# cd ..
(lfs chroot) root:/sources# rm -rf gmp-6.2.0
```

### 8.20 安装 MPFR-4.1.0

MPFR 包含用于多精度数学的函数

解压软件包

```sh
(lfs chroot) root:/sources# tar xf mpfr-4.1.0.tar.xz 
(lfs chroot) root:/sources# cd mpfr-4.1.0
```

配置并编译代码和 HTML 文档：

```sh
time { ./configure --prefix=/usr   \
            --disable-static       \
            --enable-thread-safe   \
            --docdir=/usr/share/doc/mpfr-4.1.0 && make && make html; }

# 编译完成后输出信息：
make[1]: Leaving directory '/sources/mpfr-4.1.0/tools/bench'
make[1]: Entering directory '/sources/mpfr-4.1.0'
make[1]: Nothing to be done for 'html-am'.
make[1]: Leaving directory '/sources/mpfr-4.1.0'

real    1m30.510s
user    1m9.021s
sys     0m14.586s
```

重要：本节中的 MPFR 测试套件被认为是至关重要的。在任何情况下都不要跳过它。

测试结果并确保所有测试都通过：

```sh
(lfs chroot) root:/sources/mpfr-4.1.0# time { make check; }

# 测试完成后输出信息：
============================================================================
Testsuite summary for MPFR 4.1.0
============================================================================
# TOTAL: 183
# PASS:  181
# SKIP:  2
# XFAIL: 0
# FAIL:  0
# XPASS: 0
# ERROR: 0
============================================================================
make[3]: Leaving directory '/sources/mpfr-4.1.0/tests'
make[2]: Leaving directory '/sources/mpfr-4.1.0/tests'
[tversion] MPFR 4.1.0
[tversion] Compiler: GCC 10.2.0
[tversion] C standard: __STDC__ = 1, __STDC_VERSION__ = 201710L
[tversion] __GNUC__ = 10, __GNUC_MINOR__ = 2
[tversion] __GLIBC__ = 2, __GLIBC_MINOR__ = 32
[tversion] GMP: header 6.2.0, library 6.2.0
[tversion] __GMP_CC = "gcc"
[tversion] __GMP_CFLAGS = "-O2 -pedantic -fomit-frame-pointer -m64 -mtune=skylake -march=broadwell"
[tversion] WinDLL: __GMP_LIBGMP_DLL = 0, MPFR_WIN_THREAD_SAFE_DLL = undef
[tversion] MPFR_ALLOCA_MAX = 16384
[tversion] TLS = yes, float128 = yes, decimal = no, GMP internals = no
[tversion] Shared cache = no
[tversion] intmax_t = yes, printf = yes, IEEE floats = yes
[tversion] gmp_printf: hhd = yes, lld = yes, jd = yes, td = yes, Ld = yes
[tversion] _mulx_u64 = yes
[tversion] MPFR tuning parameters from src/x86_64/mparam.h
[tversion] sizeof(long) = 8, sizeof(mpfr_intmax_t) = 8, sizeof(intmax_t) = 8
[tversion] GMP_NUMB_BITS = 64, sizeof(mp_limb_t) = 8
[tversion] Within limb: long = y/y, intmax_t = y/y
[tversion] _MPFR_PREC_FORMAT = 3, sizeof(mpfr_prec_t) = 8
[tversion] _MPFR_EXP_FORMAT = 3, sizeof(mpfr_exp_t) = 8
[tversion] sizeof(mpfr_t) = 32, sizeof(mpfr_ptr) = 8
[tversion] Precision range: [1,9223372036854775551]
[tversion] Max exponent range: [-4611686018427387903,4611686018427387903]
[tversion] Generic ABI code: no
[tversion] Enable formally proven code: no
[tversion] Locale: C
make[1]: Leaving directory '/sources/mpfr-4.1.0/tests'
Making check in tune
make[1]: Entering directory '/sources/mpfr-4.1.0/tune'
make[1]: Nothing to be done for 'check'.
make[1]: Leaving directory '/sources/mpfr-4.1.0/tune'
Making check in tools/bench
make[1]: Entering directory '/sources/mpfr-4.1.0/tools/bench'
make[1]: Nothing to be done for 'check'.
make[1]: Leaving directory '/sources/mpfr-4.1.0/tools/bench'
make[1]: Entering directory '/sources/mpfr-4.1.0'
make[1]: Nothing to be done for 'check-am'.
make[1]: Leaving directory '/sources/mpfr-4.1.0'

real    2m26.550s
user    2m3.460s
sys     0m16.751s
```

安装软件包及其文档：

```sh
(lfs chroot) root:/sources/mpfr-4.1.0# time { make install && make install-html; }

# 安装完成后输出信息：
make[1]: Leaving directory '/sources/mpfr-4.1.0/tools/bench'
make[1]: Entering directory '/sources/mpfr-4.1.0'
make[1]: Nothing to be done for 'install-html-am'.
make[1]: Leaving directory '/sources/mpfr-4.1.0'

real    0m0.900s
user    0m0.712s
sys     0m0.148s
```

清除软件包

```sh
(lfs chroot) root:/sources/mpfr-4.1.0# cd ..
(lfs chroot) root:/sources# rm -rf mpfr-4.1.0
```

### 8.21 安装 MPC-1.1.0

MPC 包含一个用于复数算术的库，具有任意高精度和正确的结果四舍五入。

解压软件包

```sh
(lfs chroot) root:/sources# tar xf mpc-1.1.0.tar.gz 
(lfs chroot) root:/sources# cd mpc-1.1.0
```

配置并编译代码和 HTML 文档：

```sh
time { ./configure --prefix=/usr  \
            --disable-static      \
            --docdir=/usr/share/doc/mpc-1.1.0 && make && make html; }

# 编译完成后输出信息：
make[1]: Entering directory '/sources/mpc-1.1.0'
make[1]: Nothing to be done for 'html-am'.
make[1]: Leaving directory '/sources/mpc-1.1.0'

real    0m24.260s
user    0m17.650s
sys     0m5.258s
```

若要测试结果，请执行：

```sh
time { make check; }

# 测试完成后输出信息：
============================================================================
Testsuite summary for mpc 1.1.0
============================================================================
# TOTAL: 67
# PASS:  67
# SKIP:  0
# XFAIL: 0
# FAIL:  0
# XPASS: 0
# ERROR: 0
============================================================================
make[3]: Leaving directory '/sources/mpc-1.1.0/tests'
make[2]: Leaving directory '/sources/mpc-1.1.0/tests'
make[1]: Leaving directory '/sources/mpc-1.1.0/tests'
Making check in doc
make[1]: Entering directory '/sources/mpc-1.1.0/doc'
make[1]: Nothing to be done for 'check'.
make[1]: Leaving directory '/sources/mpc-1.1.0/doc'
Making check in tools
make[1]: Entering directory '/sources/mpc-1.1.0/tools'
Making check in bench
make[2]: Entering directory '/sources/mpc-1.1.0/tools/bench'
make[2]: Nothing to be done for 'check'.
make[2]: Leaving directory '/sources/mpc-1.1.0/tools/bench'
make[2]: Entering directory '/sources/mpc-1.1.0/tools'
make[2]: Nothing to be done for 'check-am'.
make[2]: Leaving directory '/sources/mpc-1.1.0/tools'
make[1]: Leaving directory '/sources/mpc-1.1.0/tools'
make[1]: Entering directory '/sources/mpc-1.1.0'
make[1]: Leaving directory '/sources/mpc-1.1.0'

real    0m43.931s
user    0m37.973s
sys     0m5.095s
```

安装软件包及其文档：

```sh
time { make install && make install-html; }

# 安装完成后输出信息：
make[1]: Entering directory '/sources/mpc-1.1.0'
make[1]: Nothing to be done for 'install-html-am'.
make[1]: Leaving directory '/sources/mpc-1.1.0'

real    0m0.590s
user    0m0.380s
sys     0m0.135s
```

清除软件包

```sh
(lfs chroot) root:/sources/mpc-1.1.0# cd ..
(lfs chroot) root:/sources# rm -rf mpc-1.1.0
```

### 8.22 安装 Attr-2.4.48

attr 包含用于管理文件系统对象的扩展属性的实用程序。

解压软件包

```sh
(lfs chroot) root:/sources# tar xf attr-2.4.48.tar.gz 
(lfs chroot) root:/sources# cd attr-2.4.48
```

配置编译并检查，测试需要在支持扩展属性（例如 ext2、ext3 或 ext4 文件系统）的文件系统上运行：

```sh
time { ./configure --prefix=/usr   \
            --bindir=/bin          \
            --disable-static       \
            --sysconfdir=/etc      \
            --docdir=/usr/share/doc/attr-2.4.48 && make && make check; }

# 编译完成后输出信息：
============================================================================
Testsuite summary for attr 2.4.48
============================================================================
# TOTAL: 2
# PASS:  2
# SKIP:  0
# XFAIL: 0
# FAIL:  0
# XPASS: 0
# ERROR: 0
============================================================================
make[3]: Leaving directory '/sources/attr-2.4.48'
make[2]: Leaving directory '/sources/attr-2.4.48'
make[1]: Leaving directory '/sources/attr-2.4.48'

real    0m11.434s
user    0m6.737s
sys     0m2.488s
```

安装软件包：

```sh
time { make install; }

# 安装完成后输出信息：
make[3]: Leaving directory '/sources/attr-2.4.48'
make[2]: Leaving directory '/sources/attr-2.4.48'
make[1]: Leaving directory '/sources/attr-2.4.48'

real    0m0.740s
user    0m0.472s
sys     0m0.199s
```

共享库需要移动到 /lib，因此需要重新创建 /usr/lib 中的 .so 文件：

```sh
(lfs chroot) root:/sources/attr-2.4.48# mv -v /usr/lib/libattr.so.* /lib
renamed '/usr/lib/libattr.so.1' -> '/lib/libattr.so.1'
renamed '/usr/lib/libattr.so.1.1.2448' -> '/lib/libattr.so.1.1.2448'
(lfs chroot) root:/sources/attr-2.4.48# ln -sfv ../../lib/$(readlink /usr/lib/libattr.so) /usr/lib/libattr.so
'/usr/lib/libattr.so' -> '../../lib/libattr.so.1.1.2448'
```

清除软件包

```sh
(lfs chroot) root:/sources/attr-2.4.48# cd ..
(lfs chroot) root:/sources# rm -rf attr-2.4.48
```

### 8.23 安装 Acl-2.2.53

Acl 包含管理访问控制列表的实用程序，这些实用程序用于为文件和目录定义更细粒度的任意访问权限。

解压软件包

```sh
(lfs chroot) root:/sources# tar xf acl-2.2.53.tar.gz 
(lfs chroot) root:/sources# cd acl-2.2.53
```

配置并编译安装：

```sh
time { ./configure --prefix=/usr  \
            --bindir=/bin         \
            --disable-static      \
            --libexecdir=/usr/lib \
            --docdir=/usr/share/doc/acl-2.2.53 && make && make install; }

# 安装完成后输出信息：
make[2]: Leaving directory '/sources/acl-2.2.53'
make[1]: Leaving directory '/sources/acl-2.2.53'

real    0m16.168s
user    0m12.017s
sys     0m3.358s
```

共享库需要移动到 /lib，因此需要重新创建 /usr/lib 中的 .so 文件：

```sh
(lfs chroot) root:/sources/acl-2.2.53# mv -v /usr/lib/libacl.so.* /lib
renamed '/usr/lib/libacl.so.1' -> '/lib/libacl.so.1'
renamed '/usr/lib/libacl.so.1.1.2253' -> '/lib/libacl.so.1.1.2253'
(lfs chroot) root:/sources/acl-2.2.53# ln -sfv ../../lib/$(readlink /usr/lib/libacl.so) /usr/lib/libacl.so
'/usr/lib/libacl.so' -> '../../lib/libacl.so.1.1.2253'
```

清除软件包

```sh
(lfs chroot) root:/sources/acl-2.2.53# cd ..
(lfs chroot) root:/sources# rm -rf acl-2.2.53
```

### 8.24 安装 Libcap-2.42

Libcap 包实现了 Linux 内核中可用的 POSIX 1003.1e 功能的用户空间接口。这些功能将所有强大的 root 权限划分为一组不同的权限。

解压软件包

```sh
(lfs chroot) root:/sources# tar xf libcap-2.42.tar.xz 
(lfs chroot) root:/sources# cd libcap-2.42
```

禁止安装静态库：

```sh
(lfs chroot) root:/sources/libcap-2.42# sed -i '/install -m.*STACAPLIBNAME/d' libcap/Makefile
```

编译并测试

```sh
time { make lib=lib && make test; }

# 测试完成后输出信息：
hello libcap and libpsx .......... PASSED
make[1]: Leaving directory '/sources/libcap-2.42/tests'
make -C progs test
make[1]: Entering directory '/sources/libcap-2.42/progs'
make[1]: Nothing to be done for 'test'.
make[1]: Leaving directory '/sources/libcap-2.42/progs'

real    0m5.189s
user    0m3.744s
sys     0m0.918s
```

安装软件包

```sh
time { make lib=lib PKGCONFIGDIR=/usr/lib/pkgconfig install; }

# 完成后输出信息：
make[1]: Leaving directory '/sources/libcap-2.42/doc'
make -C kdebug install
make[1]: Entering directory '/sources/libcap-2.42/kdebug'
make[1]: Nothing to be done for 'install'.
make[1]: Leaving directory '/sources/libcap-2.42/kdebug'

real    0m0.307s
user    0m0.154s
sys     0m0.101s
```

进行一些清理

```sh
(lfs chroot) root:/sources/libcap-2.42# chmod -v 755 /lib/libcap.so.2.42
mode of '/lib/libcap.so.2.42' changed from 0644 (rw-r--r--) to 0755 (rwxr-xr-x)
(lfs chroot) root:/sources/libcap-2.42# mv -v /lib/libpsx.a /usr/lib
renamed '/lib/libpsx.a' -> '/usr/lib/libpsx.a'
(lfs chroot) root:/sources/libcap-2.42# rm -v /lib/libcap.so
removed '/lib/libcap.so'
(lfs chroot) root:/sources/libcap-2.42# ln -sfv ../../lib/libcap.so.2 /usr/lib/libcap.so'/usr/lib/libcap.so' -> '../../lib/libcap.so.2'
```

清除软件包

```sh
(lfs chroot) root:/sources/libcap-2.42# cd ..
(lfs chroot) root:/sources# rm -rf libcap-2.42
```

### 8.25 安装 Shadow-4.8.1

Shadow 软件包含以安全方式处理密码的程序。

#### 8.25.1 准备安装 Shadow

解压软件包

```sh
(lfs chroot) root:/sources# tar xf shadow-4.8.1.tar.xz 
(lfs chroot) root:/sources# cd shadow-4.8.1
```

注意：如果您想强制使用强密码，请参阅 http://www.linuxfromscratch.org/blfs/view/10.0/postlfs/cracklib.html 以在构建 Shadow 之前安装 CrackLib。然后将 --with-libcrack 添加到下面的配置命令中。

禁用组程序及其手册页的安装，因为 Coreutils 提供了更好的版本。此外，禁止安装已在第 8.3 节“Man-pages-5.08”中安装的手册页：

```sh
(lfs chroot) root:/sources/shadow-4.8.1# find man -name Makefile.in -exec sed -i 's/groups\.1 / /'   {} \;
(lfs chroot) root:/sources/shadow-4.8.1# find man -name Makefile.in -exec sed -i 's/getspnam\.3 / /' {} \;
(lfs chroot) root:/sources/shadow-4.8.1# find man -name Makefile.in -exec sed -i 's/passwd\.5 / /'   {} \;
```

不要使用默认的 crypt 方法，而是使用更安全的 SHA-512 密码加密方法，它也允许密码超过 8 个字符。还需要将 Shadow 默认使用的用户邮箱的过时 /var/spool/mail 位置更改为当前使用的 /var/mail 位置：

```sh
# 执行以下命令
sed -e 's:#ENCRYPT_METHOD DES:ENCRYPT_METHOD SHA512:' \
    -e 's:/var/spool/mail:/var/mail:'                 \
    -i etc/login.defs
```

如果您选择使用 Cracklib 支持构建 Shadow，请运行以下命令：

```sh
sed -i 's:DICTPATH.*:DICTPATH\t/lib/cracklib/pw_dict:' etc/login.defs (我不运行此代码)
```

稍作修改，使 useradd 生成的第一个组号为 1000：

```sh
(lfs chroot) root:/sources/shadow-4.8.1# sed -i 's/1000/999/' etc/useradd
```

配置并编译安装：

```sh
# 创建文件
(lfs chroot) root:/sources/shadow-4.8.1# touch /usr/bin/passwd

# 开始编译
time { ./configure --sysconfdir=/etc \
            --with-group-name-max-length=32 && make && make install; }

# 安装完成后输出信息：
make[3]: Leaving directory '/sources/shadow-4.8.1/etc'
make[2]: Leaving directory '/sources/shadow-4.8.1/etc'
make[1]: Leaving directory '/sources/shadow-4.8.1/etc'
make[1]: Entering directory '/sources/shadow-4.8.1'
make[2]: Entering directory '/sources/shadow-4.8.1'
make[2]: Nothing to be done for 'install-exec-am'.
make[2]: Nothing to be done for 'install-data-am'.
make[2]: Leaving directory '/sources/shadow-4.8.1'
make[1]: Leaving directory '/sources/shadow-4.8.1'

real    0m49.443s
user    0m35.163s
sys     0m11.691s
```

#### 8.25.2 配置 Shadow

该软件包包含用于添加、修改和删除用户和组的实用程序；设置和更改他们的密码；并执行其他管理任务。有关密码隐藏含义的完整说明，请参阅解压后的源代码树中的 `doc/HOWTO` 文件。如果使用 Shadow 支持，请记住需要验证密码的程序（显示管理器、FTP 程序、pop3 守护程序等）必须与 Shadow 兼容。也就是说，他们需要能够使用隐藏密码。

要启用影子密码，请运行以下命令：

```sh
(lfs chroot) root:/sources/shadow-4.8.1# pwconv
```

要启用影子组密码，请运行：

```sh
(lfs chroot) root:/sources/shadow-4.8.1# grpconv
```

Shadow 对 useradd 实用程序的库存配置有一些需要解释的注意事项。首先，useradd 实用程序的默认操作是创建用户和与用户同名的组。默认情况下，用户 ID (UID) 和组 ID (GID) 编号将以 1000 开头。这意味着如果您不向 useradd 传递参数，则每个用户都将成为系统上唯一组的成员。如果此行为不受欢迎，则需要将 -g 参数传递给 useradd。默认参数存储在 /etc/default/useradd 文件中。您可能需要修改此文件中的两个参数以满足您的特定需求。

#### 8.25.3 设置 root 密码

为用户 root 选择一个密码并通过运行来设置它：

```sh
(lfs chroot) root:/sources/shadow-4.8.1# passwd root
Changing password for root
Enter the new password (minimum of 5 characters)
Please use a combination of upper and lower case letters and numbers.
New password: # 这里我输入了密码 12345
Bad password: too simple.    # 由于12345是弱密码，我被要求再次输入一遍
Warning: weak password (enter it again to use it anyway).
New password: 
Re-enter new password: 
passwd: password changed.
```

清理软件包

```sh
(lfs chroot) root:/sources/shadow-4.8.1# cd ..
(lfs chroot) root:/sources# rm -rf shadow-4.8.1
```

### 8.26 安装 GCC-10.2.0

GCC 包含 GNU 编译器集合，其中包括 C 和 C++ 编译器。

‼️ 注意：GCC软件包较大，配置时间非常长 (文档说有 102SBU，**实际测试时耗大约6小时**)

请一定要耐心等待 ~~

解压软件包

```sh
(lfs chroot) root:/sources# tar xf gcc-10.2.0.tar.xz 
(lfs chroot) root:/sources# cd gcc-10.2.0
```

如果在 x86_64 上构建，请将 64 位库的默认目录名称更改为“lib”：

```sh
# 运行此代码
case $(uname -m) in
  x86_64)
    sed -e '/m64=/s/lib64/lib/' \
        -i.orig gcc/config/i386/t-linux64
  ;;
esac
```

GCC 文档建议在专用构建目录中构建 GCC：

```sh
(lfs chroot) root:/sources/gcc-10.2.0# mkdir -v build
mkdir: created directory 'build'
(lfs chroot) root:/sources/gcc-10.2.0# cd build/
```

配置并编译

```sh
time { ../configure --prefix=/usr     \
             LD=ld                    \
             --enable-languages=c,c++ \
             --disable-multilib       \
             --disable-bootstrap      \
             --with-system-zlib && make; }

# 编译完成后输出以下信息：
make[4]: Leaving directory '/sources/gcc-10.2.0/build/x86_64-pc-linux-gnu/libatomic'
make[3]: Leaving directory '/sources/gcc-10.2.0/build/x86_64-pc-linux-gnu/libatomic'
make[2]: Leaving directory '/sources/gcc-10.2.0/build/x86_64-pc-linux-gnu/libatomic'
make[1]: Leaving directory '/sources/gcc-10.2.0/build'

real    52m41.791s
user    47m36.039s
sys     3m58.435s
```

重要：在本节中，GCC 的测试套件被认为是至关重要的。在任何情况下都不要跳过它。

已知 GCC 测试套件中的一组测试会耗尽默认堆栈，因此在运行测试之前增加堆栈大小：

```sh
(lfs chroot) root:/sources/gcc-10.2.0/build# ulimit -s 32768
```

设置以非特权用户身份测试结果，但不要因错误而停止：

```sh
# 以下测试命令非常耗时！实测耗时约6小时，请耐心等待~~
(lfs chroot) root:/sources/gcc-10.2.0/build# chown -Rv tester . 
(lfs chroot) root:/sources/gcc-10.2.0/build# su tester -c "PATH=$PATH make -k check"

# 运行结束后输出以下信息：
make[3]: Leaving directory '/sources/gcc-10.2.0/build/x86_64-pc-linux-gnu/libatomic'
make[2]: Leaving directory '/sources/gcc-10.2.0/build/x86_64-pc-linux-gnu/libatomic'
make[1]: Leaving directory '/sources/gcc-10.2.0/build'
make: *** [Makefile:2350: do-check] Error 2
make: Target 'check' not remade because of errors.
(lfs chroot) root:/sources/gcc-10.2.0/build# 
```

要接收测试套件结果的摘要，请运行：

```sh
(lfs chroot) root:/sources/gcc-10.2.0/build# ../contrib/test_summary
```

如果只想看摘要，将输出用管道送至 `grep -A7 Summ`，即 `../contrib/test_summary | grep -A7 Summ`

我执行了 `../contrib/test_summary | grep -A7 Summ` 命令，输出以下内容：

```sh
(lfs chroot) root:/sources/gcc-10.2.0/build# ../contrib/test_summary | grep -A7 Summ
                === g++ Summary ===

# of expected passes            191448
# of unexpected failures        17
# of expected failures          709
# of unresolved testcases       1
# of unsupported tests          7994
/sources/gcc-10.2.0/build/gcc/xg++  version 10.2.0 (GCC) 
--
                === gcc Summary ===

# of expected passes            149288
# of unexpected failures        7
# of expected failures          695
# of unsupported tests          2221
/sources/gcc-10.2.0/build/gcc/xgcc  version 10.2.0 (GCC) 

--
                === libatomic Summary ===

# of expected passes            54
                === libgomp tests ===


Running target unix

                === libgomp Summary ===

# of expected passes            2664
# of expected failures          2
# of unsupported tests          302
                === libitm tests ===


--
                === libitm Summary ===

# of expected passes            42
# of expected failures          3
# of unsupported tests          1
                === libstdc++ tests ===


--
                === libstdc++ Summary ===

# of expected passes            14233
# of unexpected failures        7
# of expected failures          93
# of unsupported tests          343

Compiler version: 10.2.0 (GCC) 
(lfs chroot) root:/sources/gcc-10.2.0/build# 
```

结果可以与位于 https://www.linuxfromscratch.org/lfs/build-logs/10.0/i5-6600K/test-logs/824-gcc-10.2.0 的结果进行比较。

已知与 get_time 相关的六个测试失败。这些与 en_HK 语言环境有关。

此外，已知与以下文件相关的以下测试因 glibc-2.32 失败：asan_test.C、co-ret-17-void-ret-coro.C、pr95519-05-gro.C、pr80166.c。

一些意外的失败总是无法避免的。 GCC 开发人员通常知道这些问题，但尚未解决。**除非测试结果与上述 URL 中的结果有很大不同，否则可以安全地继续。**

安装软件包并删除不需要的目录：

```sh
time { make install && rm -rf /usr/lib/gcc/$(gcc -dumpmachine)/10.2.0/include-fixed/bits/; }

# 安装完成后输出以下信息：
make[4]: Nothing to be done for 'install-data-am'.
make[4]: Leaving directory '/sources/gcc-10.2.0/build/x86_64-pc-linux-gnu/libatomic'
make[3]: Leaving directory '/sources/gcc-10.2.0/build/x86_64-pc-linux-gnu/libatomic'
make[2]: Leaving directory '/sources/gcc-10.2.0/build/x86_64-pc-linux-gnu/libatomic'
make[1]: Leaving directory '/sources/gcc-10.2.0/build'

real    0m41.493s
user    0m11.331s
sys     0m25.439s
```

GCC 构建目录现在归 tester 所有，安装的头文件目录（及其内容）的所有权将不正确。将所有权更改为 root 用户和组：

```sh
(lfs chroot) root:/sources/gcc-10.2.0/build# chown -v -R root:root /usr/lib/gcc/*linux-gnu/10.2.0/include{,-fixed}
```

出于“历史”原因，创建 FHS 所需的符号链接。

```sh
(lfs chroot) root:/sources/gcc-10.2.0/build# ln -sv ../usr/bin/cpp /lib
'/lib/cpp' -> '../usr/bin/cpp'
```

添加兼容性符号链接以启用具有链接时间优化 (LTO) 的构建程序：

```sh
(lfs chroot) root:/sources/gcc-10.2.0/build# install -v -dm755 /usr/lib/bfd-plugins
install: creating directory '/usr/lib/bfd-plugins'
(lfs chroot) root:/sources/gcc-10.2.0/build# ln -sfv ../../libexec/gcc/$(gcc -dumpmachine)/10.2.0/liblto_plugin.so /usr/lib/bfd-plugins/
'/usr/lib/bfd-plugins/liblto_plugin.so' -> '../../libexec/gcc/x86_64-pc-linux-gnu/10.2.0/liblto_plugin.so'
```

现在我们的最终工具链已经到位，再次确保编译和链接按预期工作很重要。我们通过执行一些健全性检查来做到这一点：

```sh
(lfs chroot) root:/sources/gcc-10.2.0/build# echo 'int main(){}' > dummy.c
(lfs chroot) root:/sources/gcc-10.2.0/build# cc dummy.c -v -Wl,--verbose &> dummy.log
(lfs chroot) root:/sources/gcc-10.2.0/build# readelf -l a.out | grep ': /lib'
      [Requesting program interpreter: /lib64/ld-linux-x86-64.so.2]
```

如果没有错误，上一个命令的输出将是（允许动态链接器名称中特定于平台的差异）：

```sh
[Requesting program interpreter: /lib64/ld-linux-x86-64.so.2]
```

现在确保我们设置为使用正确的启动文件：

```sh
(lfs chroot) root:/sources/gcc-10.2.0/build# grep -o '/usr/lib.*/crt[1in].*succeeded' dummy.log
/usr/lib/gcc/x86_64-pc-linux-gnu/10.2.0/../../../../lib/crt1.o succeeded
/usr/lib/gcc/x86_64-pc-linux-gnu/10.2.0/../../../../lib/crti.o succeeded
/usr/lib/gcc/x86_64-pc-linux-gnu/10.2.0/../../../../lib/crtn.o succeeded
```

上一条命令的输出应该是：

```sh
/usr/lib/gcc/x86_64-pc-linux-gnu/10.2.0/../../../../lib/crt1.o succeeded
/usr/lib/gcc/x86_64-pc-linux-gnu/10.2.0/../../../../lib/crti.o succeeded
/usr/lib/gcc/x86_64-pc-linux-gnu/10.2.0/../../../../lib/crtn.o succeeded
```

根据您的机器架构，上述内容可能略有不同。不同之处在于 /usr/lib/gcc 之后的目录名称。这里要注意的重要一点是 gcc 已经在 /usr/lib 目录下找到了所有三个 crt*.o 文件。

验证编译器是否正在搜索正确的头文件：

```sh
(lfs chroot) root:/sources/gcc-10.2.0/build# grep -B4 '^ /usr/include' dummy.log
#include <...> search starts here:
 /usr/lib/gcc/x86_64-pc-linux-gnu/10.2.0/include
 /usr/local/include
 /usr/lib/gcc/x86_64-pc-linux-gnu/10.2.0/include-fixed
 /usr/include
```

此命令应返回以下输出：

```sh
#include <...> search starts here:
 /usr/lib/gcc/x86_64-pc-linux-gnu/10.2.0/include
 /usr/local/include
 /usr/lib/gcc/x86_64-pc-linux-gnu/10.2.0/include-fixed
 /usr/include
```

同样，以您的目标三元组命名的目录可能与上述不同，具体取决于您的系统架构。

接下来，验证新链接器是否与正确的搜索路径一起使用：

```sh
(lfs chroot) root:/sources/gcc-10.2.0/build# grep 'SEARCH.*/usr/lib' dummy.log |sed 's|; |\n|g'
SEARCH_DIR("/usr/x86_64-pc-linux-gnu/lib64")
SEARCH_DIR("/usr/local/lib64")
SEARCH_DIR("/lib64")
SEARCH_DIR("/usr/lib64")
SEARCH_DIR("/usr/x86_64-pc-linux-gnu/lib")
SEARCH_DIR("/usr/local/lib")
SEARCH_DIR("/lib")
SEARCH_DIR("/usr/lib");
```

对包含带有 `-linux-gnu` 组件的路径的引用应该被忽略，上一个命令的输出应该是：

```sh
SEARCH_DIR("/usr/x86_64-pc-linux-gnu/lib64")
SEARCH_DIR("/usr/local/lib64")
SEARCH_DIR("/lib64")
SEARCH_DIR("/usr/lib64")
SEARCH_DIR("/usr/x86_64-pc-linux-gnu/lib")
SEARCH_DIR("/usr/local/lib")
SEARCH_DIR("/lib")
SEARCH_DIR("/usr/lib");
```

32 位系统可能会看到几个不同的目录。例如，这是 i686 机器的输出：

```sh
SEARCH_DIR("/usr/i686-pc-linux-gnu/lib32")
SEARCH_DIR("/usr/local/lib32")
SEARCH_DIR("/lib32")
SEARCH_DIR("/usr/lib32")
SEARCH_DIR("/usr/i686-pc-linux-gnu/lib")
SEARCH_DIR("/usr/local/lib")
SEARCH_DIR("/lib")
SEARCH_DIR("/usr/lib");
```

接下来确保我们使用了正确的 libc：

```sh
(lfs chroot) root:/sources/gcc-10.2.0/build# grep "/lib.*/libc.so.6 " dummy.log
attempt to open /lib/libc.so.6 succeeded
```

上一条命令的输出应该是：

```sh
attempt to open /lib/libc.so.6 succeeded
```

确保 GCC 使用正确的动态链接器：

```sh
(lfs chroot) root:/sources/gcc-10.2.0/build# grep found dummy.log
found ld-linux-x86-64.so.2 at /lib/ld-linux-x86-64.so.2
```

上一个命令的输出应该是（允许动态链接器名称中特定于平台的差异）：

```sh
found ld-linux-x86-64.so.2 at /lib/ld-linux-x86-64.so.2
```

**如果输出未如上所示出现或根本没有收到，则说明出现了严重错误**。调查并追溯这些步骤以找出问题所在并进行纠正。在继续该过程之前，需要解决任何问题。

一切正常后，清理测试文件：

```sh
(lfs chroot) root:/sources/gcc-10.2.0/build# rm -v dummy.c a.out dummy.log
removed 'dummy.c'
removed 'a.out'
removed 'dummy.log'
```

最后，移动一个错位的文件：

```sh
(lfs chroot) root:/sources/gcc-10.2.0/build# mkdir -pv /usr/share/gdb/auto-load/usr/lib
mkdir: created directory '/usr/share/gdb'
mkdir: created directory '/usr/share/gdb/auto-load'
mkdir: created directory '/usr/share/gdb/auto-load/usr'
mkdir: created directory '/usr/share/gdb/auto-load/usr/lib'
(lfs chroot) root:/sources/gcc-10.2.0/build# mv -v /usr/lib/*gdb.py /usr/share/gdb/auto-load/usr/lib
renamed '/usr/lib/libstdc++.so.6.0.28-gdb.py' -> '/usr/share/gdb/auto-load/usr/lib/libstdc++.so.6.0.28-gdb.py'
```

清除软件包

```sh
(lfs chroot) root:/sources/gcc-10.2.0/build# cd ../..
(lfs chroot) root:/sources# rm -rf gcc-10.2.0
```

### 8.27 安装 Pkg-config-0.29.2

pkg-config 包含一个工具，用于在包安装的配置和制作阶段将包含路径和/或库路径传递给构建工具。

解压软件包

```sh
(lfs chroot) root:/sources# tar xf pkg-config-0.29.2.tar.gz 
(lfs chroot) root:/sources# cd pkg-config-0.29.2
```

配置编译并检查

```sh
time { ./configure --prefix=/usr       \
            --with-internal-glib       \
            --disable-host-tool        \
            --docdir=/usr/share/doc/pkg-config-0.29.2 && make && make check; }

# 检查完毕后输出以下信息：
===================
All 30 tests passed
===================
make[2]: Leaving directory '/sources/pkg-config-0.29.2/check'
make[1]: Leaving directory '/sources/pkg-config-0.29.2/check'

real    1m22.676s
user    1m1.624s
sys     0m15.836s
```

安装软件包

```sh
time { make install; }

# 安装完毕后输出以下信息：
make[2]: Nothing to be done for 'install-data-am'.
make[2]: Leaving directory '/sources/pkg-config-0.29.2/check'
make[1]: Leaving directory '/sources/pkg-config-0.29.2/check'

real    0m0.799s
user    0m0.510s
sys     0m0.179s
```

清除软件包

```sh
(lfs chroot) root:/sources/pkg-config-0.29.2# cd ..
(lfs chroot) root:/sources# rm -rf pkg-config-0.29.2
```

### 8.28 安装 Ncurses-6.2

Ncurses 包含用于独立于终端处理字符屏幕的库。

解压软件包

```sh
(lfs chroot) root:/sources# tar xf ncurses-6.2.tar.gz 
(lfs chroot) root:/sources# cd ncurses-6.2
```

禁止安装不由 configure 处理的静态库：

```sh
(lfs chroot) root:/sources/ncurses-6.2# sed -i '/LIBTOOL_INSTALL/d' c++/Makefile.in
```

配置并编译安装

```sh
time { ./configure --prefix=/usr    \
            --mandir=/usr/share/man \
            --with-shared           \
            --without-debug         \
            --without-normal        \
            --enable-pc-files       \
            --enable-widec && make && make install; }

# 安装完成后输出信息：
installing ./cursesapp.h in /usr/include
installing ./cursesf.h in /usr/include
installing ./cursesm.h in /usr/include
installing ./cursesp.h in /usr/include
installing ./cursesw.h in /usr/include
installing ./cursslk.h in /usr/include
installing etip.h in /usr/include
make[1]: Leaving directory '/sources/ncurses-6.2/c++'

real    1m47.320s
user    1m17.706s
sys     0m19.340s
```

将共享库移动到 /lib 目录，它们应该驻留在那里：

```sh
(lfs chroot) root:/sources/ncurses-6.2# mv -v /usr/lib/libncursesw.so.6* /lib
renamed '/usr/lib/libncursesw.so.6' -> '/lib/libncursesw.so.6'
renamed '/usr/lib/libncursesw.so.6.2' -> '/lib/libncursesw.so.6.2'
```

由于库已被移动，一个符号链接指向一个不存在的文件。重新创建它：

```sh
(lfs chroot) root:/sources/ncurses-6.2# ln -sfv ../../lib/$(readlink /usr/lib/libncursesw.so) /usr/lib/libncursesw.so
'/usr/lib/libncursesw.so' -> '../../lib/libncursesw.so.6'
```

许多应用程序仍然希望链接器能够找到非宽字符 Ncurses 库。通过符号链接和链接器脚本，诱使此类应用程序与宽字符库链接：

```sh
# 执行以下脚本
for lib in ncurses form panel menu ; do
    rm -vf                    /usr/lib/lib${lib}.so
    echo "INPUT(-l${lib}w)" > /usr/lib/lib${lib}.so
    ln -sfv ${lib}w.pc        /usr/lib/pkgconfig/${lib}.pc
done

# 执行之后的输出信息 (不要重复执行以下代码)：
(lfs chroot) root:/sources/ncurses-6.2# for lib in ncurses form panel menu ; do
>     rm -vf                    /usr/lib/lib${lib}.so
>     echo "INPUT(-l${lib}w)" > /usr/lib/lib${lib}.so
>     ln -sfv ${lib}w.pc        /usr/lib/pkgconfig/${lib}.pc
> done
removed '/usr/lib/libncurses.so'
'/usr/lib/pkgconfig/ncurses.pc' -> 'ncursesw.pc'
'/usr/lib/pkgconfig/form.pc' -> 'formw.pc'
'/usr/lib/pkgconfig/panel.pc' -> 'panelw.pc'
'/usr/lib/pkgconfig/menu.pc' -> 'menuw.pc'
```

最后，确保在构建时寻找 `-lcurses` 的旧应用程序仍然可以构建：

```sh
(lfs chroot) root:/sources/ncurses-6.2# rm -vf /usr/lib/libcursesw.so
(lfs chroot) root:/sources/ncurses-6.2# echo "INPUT(-lncursesw)" > /usr/lib/libcursesw.so
(lfs chroot) root:/sources/ncurses-6.2# ln -sfv libncurses.so /usr/lib/libcurses.so
'/usr/lib/libcurses.so' -> 'libncurses.so'
```

如果需要，请安装 Ncurses 文档：

```sh
(lfs chroot) root:/sources/ncurses-6.2# mkdir -v /usr/share/doc/ncurses-6.2
mkdir: created directory '/usr/share/doc/ncurses-6.2'
(lfs chroot) root:/sources/ncurses-6.2# cp -v -R doc/* /usr/share/doc/ncurses-6.2
```

注意：以上代码说明不会创建非宽字符 Ncurses 库，因为从源代码编译安装的任何包都不会在运行时链接到它们。但是，唯一已知的与非宽字符 Ncurses 库链接的纯二进制应用程序需要版本 5。如果由于某些纯二进制应用程序或要与 LSB 兼容而必须拥有此类库，请使用以下命令再次构建包：

```sh
# 执行以下命令
make distclean
./configure --prefix=/usr    \
            --with-shared    \
            --without-normal \
            --without-debug  \
            --without-cxx-binding \
            --with-abi-version=5 
make sources libs
cp -av lib/lib*.so.5* /usr/lib

# 最后一条命令执行的结果如下：
'lib/libform.so.5' -> '/usr/lib/libform.so.5'
'lib/libform.so.5.9' -> '/usr/lib/libform.so.5.9'
'lib/libmenu.so.5' -> '/usr/lib/libmenu.so.5'
'lib/libmenu.so.5.9' -> '/usr/lib/libmenu.so.5.9'
'lib/libncurses.so.5' -> '/usr/lib/libncurses.so.5'
'lib/libncurses.so.5.9' -> '/usr/lib/libncurses.so.5.9'
'lib/libpanel.so.5' -> '/usr/lib/libpanel.so.5'
'lib/libpanel.so.5.9' -> '/usr/lib/libpanel.so.5.9'
```

清除软件包

```sh
(lfs chroot) root:/sources/ncurses-6.2# cd ..
(lfs chroot) root:/sources# rm -rf ncurses-6.2
```

### 8.29 安装 Sed-4.8

Sed 包含一个流编辑器。

解压软件包

```sh
(lfs chroot) root:/sources# tar xf sed-4.8.tar.xz 
(lfs chroot) root:/sources# cd sed-4.8
```

配置并编译代码和 HTML 文档

```sh
time { ./configure --prefix=/usr --bindir=/bin && make && make html; }

# 编译完成后输出以下内容：
make[2]: Entering directory '/sources/sed-4.8/gnulib-tests'
make[2]: Nothing to be done for 'html-am'.
make[2]: Leaving directory '/sources/sed-4.8/gnulib-tests'
make[1]: Leaving directory '/sources/sed-4.8/gnulib-tests'

real    0m52.683s
user    0m32.202s
sys     0m11.508s
```

若要进行测试，请执行：

```sh
(lfs chroot) root:/sources/sed-4.8# chown -Rv tester .
(lfs chroot) root:/sources/sed-4.8# su tester -c "PATH=$PATH make check"

# 最后一条命令的执行结果：
============================================================================
Testsuite summary for GNU sed 4.8
============================================================================
# TOTAL: 178
# PASS:  157
# SKIP:  21
# XFAIL: 0
# FAIL:  0
# XPASS: 0
# ERROR: 0
============================================================================
make[6]: Leaving directory '/sources/sed-4.8/gnulib-tests'
make[5]: Leaving directory '/sources/sed-4.8/gnulib-tests'
make[4]: Leaving directory '/sources/sed-4.8/gnulib-tests'
make[3]: Leaving directory '/sources/sed-4.8/gnulib-tests'
make[2]: Leaving directory '/sources/sed-4.8/gnulib-tests'
make[1]: Leaving directory '/sources/sed-4.8'
```

安装软件包及其文档：

```sh
(lfs chroot) root:/sources/sed-4.8# make install
(lfs chroot) root:/sources/sed-4.8# install -d -m755 /usr/share/doc/sed-4.8
(lfs chroot) root:/sources/sed-4.8# install -m644 doc/sed.html /usr/share/doc/sed-4.8
```

清除软件包

```sh
(lfs chroot) root:/sources/sed-4.8# cd ..
(lfs chroot) root:/sources# rm -rf sed-4.8
```

### 8.30 安装 Psmisc-23.3

Psmisc 包含用于显示有关正在运行进程的信息的程序。

解压软件包

```sh
(lfs chroot) root:/sources# tar xf psmisc-23.3.tar.xz 
(lfs chroot) root:/sources# cd psmisc-23.3
```

配置并编译安装

```sh
time { ./configure --prefix=/usr && make && make install; }

# 安装完成后输出信息：
make[4]: Leaving directory '/sources/psmisc-23.3'
make[3]: Nothing to be done for 'install-data-am'.
make[3]: Leaving directory '/sources/psmisc-23.3'
make[2]: Leaving directory '/sources/psmisc-23.3'
make[1]: Leaving directory '/sources/psmisc-23.3'

real    0m13.287s
user    0m8.342s
sys     0m3.418s
```

最后，将 killall 和 fuser 程序移动到 FHS 指定的位置：

```sh
(lfs chroot) root:/sources/psmisc-23.3# mv -v /usr/bin/fuser /bin
renamed '/usr/bin/fuser' -> '/bin/fuser'
(lfs chroot) root:/sources/psmisc-23.3# mv -v /usr/bin/killall /bin
renamed '/usr/bin/killall' -> '/bin/killall'
```

清除软件包

```sh
(lfs chroot) root:/sources/psmisc-23.3# cd ..
(lfs chroot) root:/sources# rm -rf psmisc-23.3
```

### 8.31 安装 Gettext-0.21

Gettext 包含用于国际化和本地化的实用程序。这些允许使用 NLS（本机语言支持）编译程序，使它们能够以用户的母语输出消息。

解压软件包

```sh
(lfs chroot) root:/sources# tar xf gettext-0.21.tar.xz 
(lfs chroot) root:/sources# cd gettext-0.21
```

配置编译并检查

```sh
time { ./configure --prefix=/usr  \
            --disable-static      \
            --docdir=/usr/share/doc/gettext-0.21 && make && make check; }

# 检查完成后输出信息：
============================================================================
Testsuite summary for gettext-tools 0.21
============================================================================
# TOTAL: 266
# PASS:  252
# SKIP:  14
# XFAIL: 0
# FAIL:  0
# XPASS: 0
# ERROR: 0
============================================================================
make[7]: Leaving directory '/sources/gettext-0.21/gettext-tools/gnulib-tests'
make[6]: Leaving directory '/sources/gettext-0.21/gettext-tools/gnulib-tests'
make[5]: Leaving directory '/sources/gettext-0.21/gettext-tools/gnulib-tests'
make[4]: Leaving directory '/sources/gettext-0.21/gettext-tools/gnulib-tests'
make[3]: Leaving directory '/sources/gettext-0.21/gettext-tools/gnulib-tests'
Making check in examples
make[3]: Entering directory '/sources/gettext-0.21/gettext-tools/examples'
Making check in po
make[4]: Entering directory '/sources/gettext-0.21/gettext-tools/examples/po'
make[4]: Nothing to be done for 'check'.
make[4]: Leaving directory '/sources/gettext-0.21/gettext-tools/examples/po'
make[4]: Entering directory '/sources/gettext-0.21/gettext-tools/examples'
make[4]: Nothing to be done for 'check-am'.
make[4]: Leaving directory '/sources/gettext-0.21/gettext-tools/examples'
make[3]: Leaving directory '/sources/gettext-0.21/gettext-tools/examples'
Making check in doc
make[3]: Entering directory '/sources/gettext-0.21/gettext-tools/doc'
make[3]: Nothing to be done for 'check'.
make[3]: Leaving directory '/sources/gettext-0.21/gettext-tools/doc'
make[3]: Entering directory '/sources/gettext-0.21/gettext-tools'
make[3]: Leaving directory '/sources/gettext-0.21/gettext-tools'
make[2]: Leaving directory '/sources/gettext-0.21/gettext-tools'
make[2]: Entering directory '/sources/gettext-0.21'
make[2]: Nothing to be done for 'check-am'.
make[2]: Leaving directory '/sources/gettext-0.21'
make[1]: Leaving directory '/sources/gettext-0.21'

real    12m59.211s
user    9m42.016s
sys     2m52.736s
```

安装软件包

```sh
time { make install; }

# 安装完成后输出信息：
make[4]: Leaving directory '/sources/gettext-0.21/gettext-tools'
make[3]: Leaving directory '/sources/gettext-0.21/gettext-tools'
make[2]: Leaving directory '/sources/gettext-0.21/gettext-tools'
make[2]: Entering directory '/sources/gettext-0.21'
make[3]: Entering directory '/sources/gettext-0.21'
make[3]: Nothing to be done for 'install-exec-am'.
make[3]: Nothing to be done for 'install-data-am'.
make[3]: Leaving directory '/sources/gettext-0.21'
make[2]: Leaving directory '/sources/gettext-0.21'
make[1]: Leaving directory '/sources/gettext-0.21'

real    0m30.462s
user    0m16.538s
sys     0m8.840s
```

执行最后操作

```sh
(lfs chroot) root:/sources/gettext-0.21# chmod -v 0755 /usr/lib/preloadable_libintl.so
mode of '/usr/lib/preloadable_libintl.so' changed from 0644 (rw-r--r--) to 0755 (rwxr-xr-x)
```

清除软件包

```sh
(lfs chroot) root:/sources/gettext-0.21# cd ..
(lfs chroot) root:/sources# rm -rf gettext-0.21
```

### 8.32 安装 Bison-3.7.1

Bison 包含一个解析器生成器。

解压软件包

```sh
(lfs chroot) root:/sources# tar xf bison-3.7.1.tar.xz 
(lfs chroot) root:/sources# cd bison-3.7.1
```

配置编译并检查

```sh
time { ./configure --prefix=/usr --docdir=/usr/share/doc/bison-3.7.1 && make && make check; }

# 检查完成后输出信息：
## ------------- ##
## Test results. ##
## ------------- ##

617 tests were successful.
43 tests were skipped.
make[3]: Leaving directory '/sources/bison-3.7.1'
make[2]: Leaving directory '/sources/bison-3.7.1'
make[1]: Leaving directory '/sources/bison-3.7.1'

real    21m30.589s
user    19m8.776s
sys     2m5.324s
```

安装软件包

```sh
time { make install; }

# 安装完成后输出信息：
make[3]: Leaving directory '/sources/bison-3.7.1'
make[2]: Leaving directory '/sources/bison-3.7.1'
make[1]: Leaving directory '/sources/bison-3.7.1'

real    0m2.989s
user    0m1.614s
sys     0m0.891s
```

清除软件包

```sh
(lfs chroot) root:/sources/bison-3.7.1# cd ..
(lfs chroot) root:/sources# rm -rf bison-3.7.1
```

### 8.33 安装 Grep-3.4

Grep 包含用于搜索文件内容的程序。

解压软件包

```sh
(lfs chroot) root:/sources# tar xf grep-3.4.tar.xz 
(lfs chroot) root:/sources# cd grep-3.4
```

配置编译并检查

```sh
time { ./configure --prefix=/usr --bindir=/bin && make && make check; }

# 检查完成后输出信息：
============================================================================
Testsuite summary for GNU grep 3.4
============================================================================
# TOTAL: 185
# PASS:  177
# SKIP:  8
# XFAIL: 0
# FAIL:  0
# XPASS: 0
# ERROR: 0
============================================================================
make[5]: Leaving directory '/sources/grep-3.4/gnulib-tests'
make[4]: Leaving directory '/sources/grep-3.4/gnulib-tests'
make[3]: Leaving directory '/sources/grep-3.4/gnulib-tests'
make[2]: Leaving directory '/sources/grep-3.4/gnulib-tests'
make[1]: Leaving directory '/sources/grep-3.4/gnulib-tests'
make[1]: Entering directory '/sources/grep-3.4'
make[1]: Leaving directory '/sources/grep-3.4'
  GEN      public-submodule-commit

real    2m31.568s
user    1m49.638s
sys     0m28.150s
```

安装软件包

```sh
time { make install; }

# 安装完成后输出信息：
make[2]: Nothing to be done for 'install-data-am'.
make[2]: Leaving directory '/sources/grep-3.4'
make[1]: Leaving directory '/sources/grep-3.4'

real    0m1.538s
user    0m1.037s
sys     0m0.323s
```

清除软件包

```sh
(lfs chroot) root:/sources/grep-3.4# cd ..
(lfs chroot) root:/sources# rm -rf grep-3.4
```

### 8.34 安装 Bash-5.0

解压软件包

```sh
(lfs chroot) root:/sources# tar xf bash-5.0.tar.gz 
(lfs chroot) root:/sources# cd bash-5.0
```

合并一些上游开发者的修复：

```sh
(lfs chroot) root:/sources/bash-5.0# patch -Np1 -i ../bash-5.0-upstream_fixes-1.patch
patching file bashhist.c
patching file bashline.c
patching file builtins/evalstring.c
patching file builtins/shopt.def
patching file command.h
patching file execute_cmd.c
patching file jobs.c
patching file lib/glob/glob.c
patching file lib/glob/glob.h
patching file lib/glob/glob_loop.c
patching file lib/readline/histfile.c
patching file parser.h
patching file parse.y
patching file patchlevel.h
patching file pathexp.c
patching file subst.c
patching file tests/varenv.right
patching file variables.c
patching file y.tab.c
```

准备配置并编译

```sh
time { ./configure --prefix=/usr             \
            --docdir=/usr/share/doc/bash-5.0 \
            --without-bash-malloc            \
            --with-installed-readline && make; }

# 编译完成后输出的信息：
-rwxr-xr-x 1 root root 3786904 Dec 22 20:48 bash
size bash
   text    data     bss     dec     hex filename
 866879   15832   55832  938543   e522f bash
make[1]: Entering directory '/sources/bash-5.0/support'
rm -f man2html.o
gcc -c  -DHAVE_CONFIG_H -DSHELL  -I/sources/bash-5.0 -I..   -g -O2 man2html.c
gcc  -DHAVE_CONFIG_H -DSHELL  -I/sources/bash-5.0 -I..   -g -O2 man2html.o -o man2html -ldl 
make[1]: Leaving directory '/sources/bash-5.0/support'

real    1m23.054s
user    0m58.441s
sys     0m13.810s
```

要准备测试，请确保 tester 用户可以写入源代码树：

```sh
(lfs chroot) root:/sources/bash-5.0# chown -Rv tester .
```

现在，以 tester 用户身份运行测试：

```sh
su tester << EOF
PATH=$PATH make tests < $(tty)
EOF

# 测试完毕后输出的一些信息：
run-type
run-varenv
run-vredir
14,16d13
< bad foo 1
< bad foo 2
< bad foo 3
```

安装软件包并将主可执行文件移动到 /bin：

```sh
time { make install && mv -vf /usr/bin/bash /bin; }

# 安装完成后输出的信息：
make[1]: Leaving directory '/sources/bash-5.0/examples/loadables'
renamed '/usr/bin/bash' -> '/bin/bash'

real    0m5.608s
user    0m3.975s
sys     0m1.267s
```

运行新编译的 bash 程序（替换当前正在执行的程序）：

```sh
(lfs chroot) root:/sources/bash-5.0# exec /bin/bash --login +h
```

上述命令使用的参数使 bash 进程成为交互式登录 shell，并继续禁用散列，以便在新程序可用时找到它们。

清除软件包

```sh
(lfs chroot) root:/sources/bash-5.0# cd ..
(lfs chroot) root:/sources# rm -rf bash-5.0
```

### 8.35 安装 Libtool-2.4.6

Libtool 包包含 GNU 通用库支持脚本。它将使用共享库的复杂性包装在一个一致的、可移植的界面中。

解压软件包

```sh
(lfs chroot) root:/sources# tar xf libtool-2.4.6.tar.xz 
(lfs chroot) root:/sources# cd libtool-2.4.6
```

配置编译并检查，注意在具有多核的系统上，可以显著减少 libtool 的测试时间。为此，请将 `TESTSUITEFLAGS=-j<N>` 附加到上面的行。例如，使用 `-j4` 可以将测试时间减少 60% 以上。

```sh
time { ./configure --prefix=/usr && make && make -j4 check; }

# 检查完成后输出信息：
## ------------- ##
## Test results. ##
## ------------- ##

ERROR: 139 tests were run,
64 failed (59 expected failures).
31 tests were skipped.
## -------------------------- ##
## testsuite.log was created. ##
## -------------------------- ##

Please send `tests/testsuite.log' and all information you think might help:

   To: <bug-libtool@gnu.org>
   Subject: [GNU Libtool 2.4.6] testsuite: 123 124 125 126 130 failed

You may investigate any problem if you feel able to do so, in which
case the test suite provides a good starting point.  Its output may
be found below `tests/testsuite.dir'.

make[3]: *** [Makefile:2459: check-local] Error 1
make[3]: Leaving directory '/sources/libtool-2.4.6'
make[2]: *** [Makefile:1897: check-am] Error 2
make[2]: Leaving directory '/sources/libtool-2.4.6'
make[1]: *** [Makefile:1606: check-recursive] Error 1
make[1]: Leaving directory '/sources/libtool-2.4.6'
make: *** [Makefile:1899: check] Error 2

real    7m57.093s
user    6m10.914s
sys     2m2.160s
(lfs chroot) root:/sources/libtool-2.4.6# 
```

可以看到，我在139项测试中，有64项失败了。我们暂时忽略这些问题。

由于循环依赖，已知有五个测试会在 LFS 构建环境中失败，但如果在安装 automake 后重新检查，所有测试都会通过。

安装软件包

```sh
time { make install; }

# 安装完成后输出信息：
make[3]: Leaving directory '/sources/libtool-2.4.6'
make[2]: Leaving directory '/sources/libtool-2.4.6'
make[1]: Leaving directory '/sources/libtool-2.4.6'

real    0m1.204s
user    0m0.753s
sys     0m0.352s
```

清除软件包

```sh
(lfs chroot) root:/sources/libtool-2.4.6# cd ..
(lfs chroot) root:/sources# rm -rf libtool-2.4.6
```

### 8.36 安装 GDBM-1.18.1

GDBM 包包含 GNU 数据库管理器。它是一个使用可扩展散列的数据库函数库，其工作方式类似于标准的 UNIX dbm。该库提供了用于存储密钥/数据对、通过密钥搜索和检索数据以及删除密钥及其数据的原语。

解压软件包

```sh
(lfs chroot) root:/sources# tar xf gdbm-1.18.1.tar.gz 
(lfs chroot) root:/sources# cd gdbm-1.18.1
```

修复在 gcc-10 上首次发现的问题：

```sh
(lfs chroot) root:/sources/gdbm-1.18.1# sed -r -i '/^char.*parseopt_program_(doc|args)/d' src/parseopt.c
```

配置编译并检查

```sh
time { ./configure --prefix=/usr  \
            --disable-static      \
            --enable-libgdbm-compat && make && make check; }

# 检查完成后输出信息：
## ------------- ##
## Test results. ##
## ------------- ##

All 30 tests were successful.
make[2]: Leaving directory '/sources/gdbm-1.18.1/tests'
make[1]: Leaving directory '/sources/gdbm-1.18.1/tests'
make[1]: Entering directory '/sources/gdbm-1.18.1'
make[1]: Leaving directory '/sources/gdbm-1.18.1'

real    0m28.880s
user    0m19.805s
sys     0m6.527s
```

安装软件包

```sh
time { make install; }

# 安装完成后输出信息：
make[1]: Leaving directory '/sources/gdbm-1.18.1/tests'
make[1]: Entering directory '/sources/gdbm-1.18.1'
make[2]: Entering directory '/sources/gdbm-1.18.1'
make[2]: Nothing to be done for 'install-exec-am'.
make[2]: Nothing to be done for 'install-data-am'.
make[2]: Leaving directory '/sources/gdbm-1.18.1'
make[1]: Leaving directory '/sources/gdbm-1.18.1'

real    0m1.264s
user    0m0.740s
sys     0m0.297s
```

清除软件包

```sh
(lfs chroot) root:/sources/gdbm-1.18.1# cd ..
(lfs chroot) root:/sources# rm -rf gdbm-1.18.1
```

### 8.37 安装 Gperf-3.1

Gperf 从一个键集生成一个完美的散列函数。

解压软件包

```sh
(lfs chroot) root:/sources# tar xf gperf-3.1.tar.gz 
(lfs chroot) root:/sources# cd gperf-3.1
```

配置编译并检查，如果同时运行多个测试（-j 选项大于 1），则测试会失败。要测试结果，请执行：

```sh
time { ./configure --prefix=/usr --docdir=/usr/share/doc/gperf-3.1 && make && make -j1 check; }

# 检查完成后输出信息：
make[1]: Leaving directory '/sources/gperf-3.1/tests'
cd doc; make check
make[1]: Entering directory '/sources/gperf-3.1/doc'
make[1]: Nothing to be done for 'check'.
make[1]: Leaving directory '/sources/gperf-3.1/doc'

real    0m12.109s
user    0m7.916s
sys     0m2.965s
```

安装软件包

```sh
time { make install; }

# 安装完成后输出信息：
mkdir -p -- /usr/share/doc/gperf-3.1
/usr/bin/install -c -m 644 ./gperf.html /usr/share/doc/gperf-3.1/gperf.html
make[1]: Leaving directory '/sources/gperf-3.1/doc'

real    0m0.099s
user    0m0.044s
sys     0m0.025s
```

清除软件包

```sh
(lfs chroot) root:/sources/gperf-3.1# cd ..
(lfs chroot) root:/sources# rm -rf gperf-3.1
```

### 8.38 安装 Expat-2.2.9

Expat 包含一个用于解析 XML 的面向流的 C 库。

解压软件包

```sh
(lfs chroot) root:/sources# tar xf expat-2.2.9.tar.xz 
(lfs chroot) root:/sources# cd expat-2.2.9
```

配置编译并检查：

```sh
time { ./configure --prefix=/usr    \
            --disable-static        \
            --docdir=/usr/share/doc/expat-2.2.9 && make && make check; }

# 检查完成后输出信息：
============================================================================
Testsuite summary for expat 2.2.9
============================================================================
# TOTAL: 2
# PASS:  2
# SKIP:  0
# XFAIL: 0
# FAIL:  0
# XPASS: 0
# ERROR: 0
============================================================================
make[4]: Leaving directory '/sources/expat-2.2.9/tests'
make[3]: Leaving directory '/sources/expat-2.2.9/tests'
make[2]: Leaving directory '/sources/expat-2.2.9/tests'
Making check in benchmark
make[2]: Entering directory '/sources/expat-2.2.9/tests/benchmark'
make[2]: Nothing to be done for 'check'.
make[2]: Leaving directory '/sources/expat-2.2.9/tests/benchmark'
make[1]: Leaving directory '/sources/expat-2.2.9/tests'
Making check in xmlwf
make[1]: Entering directory '/sources/expat-2.2.9/xmlwf'
make[1]: Nothing to be done for 'check'.
make[1]: Leaving directory '/sources/expat-2.2.9/xmlwf'
Making check in doc
make[1]: Entering directory '/sources/expat-2.2.9/doc'
make[1]: Nothing to be done for 'check'.
make[1]: Leaving directory '/sources/expat-2.2.9/doc'
make[1]: Entering directory '/sources/expat-2.2.9'
make[1]: Leaving directory '/sources/expat-2.2.9'

real    0m32.063s
user    0m25.939s
sys     0m4.536s
```

安装软件包

```sh
time { make install; }

# 安装完成后输出信息：
make[2]: Leaving directory '/sources/expat-2.2.9'
make[1]: Leaving directory '/sources/expat-2.2.9'

real    0m0.577s
user    0m0.371s
sys     0m0.104s
```

如果需要，请安装文档：

```sh
(lfs chroot) root:/sources/expat-2.2.9# install -v -m644 doc/*.{html,png,css} /usr/share/doc/expat-2.2.9
'doc/reference.html' -> '/usr/share/doc/expat-2.2.9/reference.html'
'doc/expat.png' -> '/usr/share/doc/expat-2.2.9/expat.png'
'doc/valid-xhtml10.png' -> '/usr/share/doc/expat-2.2.9/valid-xhtml10.png'
'doc/style.css' -> '/usr/share/doc/expat-2.2.9/style.css'
```

清除软件包

```sh
(lfs chroot) root:/sources/expat-2.2.9# cd .. 
(lfs chroot) root:/sources# rm -rf expat-2.2.9
```

### 8.39 安装 Inetutils-1.9.4

Inetutils 包含用于基本网络的程序。

解压软件包

```sh
(lfs chroot) root:/sources# tar xf inetutils-1.9.4.tar.xz 
(lfs chroot) root:/sources# cd inetutils-1.9.4
```

配置编译并检查：

```sh
time { ./configure --prefix=/usr \
            --localstatedir=/var \
            --disable-logger     \
            --disable-whois      \
            --disable-rcp        \
            --disable-rexec      \
            --disable-rlogin     \
            --disable-rsh        \
            --disable-servers && make && make check; }

# 检查完成后输出信息：
============================================================================
Testsuite summary for GNU inetutils 1.9.4
============================================================================
# TOTAL: 10
# PASS:  10
# SKIP:  0
# XFAIL: 0
# FAIL:  0
# XPASS: 0
# ERROR: 0
============================================================================
make[5]: Leaving directory '/sources/inetutils-1.9.4/tests'
make[4]: Leaving directory '/sources/inetutils-1.9.4/tests'
make[3]: Leaving directory '/sources/inetutils-1.9.4/tests'
make[2]: Leaving directory '/sources/inetutils-1.9.4/tests'
make[2]: Entering directory '/sources/inetutils-1.9.4'
make[2]: Leaving directory '/sources/inetutils-1.9.4'
make[1]: Leaving directory '/sources/inetutils-1.9.4'

real    1m46.111s
user    1m1.221s
sys     0m26.128s
```

注意：测试 `libls.sh` 在初始 chroot 环境中可能会失败，但如果在 LFS 系统完成后重新运行该测试，它就会通过。如果主机系统没有 ipv6 功能，一项测试 `ping-localhost.sh` 将失败。

安装软件包

```sh
time { make install; }

# 安装完成后输出信息：
make[3]: Entering directory '/sources/inetutils-1.9.4'
make[3]: Nothing to be done for 'install-exec-am'.
make[3]: Nothing to be done for 'install-data-am'.
make[3]: Leaving directory '/sources/inetutils-1.9.4'
make[2]: Leaving directory '/sources/inetutils-1.9.4'
make[1]: Leaving directory '/sources/inetutils-1.9.4'

real    0m2.032s
user    0m1.450s
sys     0m0.386s
```

移动一些程序，以便在 /usr 不可访问时它们可用：

```sh
(lfs chroot) root:/sources/inetutils-1.9.4# mv -v /usr/bin/{hostname,ping,ping6,traceroute} /bin
renamed '/usr/bin/hostname' -> '/bin/hostname'
renamed '/usr/bin/ping' -> '/bin/ping'
renamed '/usr/bin/ping6' -> '/bin/ping6'
renamed '/usr/bin/traceroute' -> '/bin/traceroute'
(lfs chroot) root:/sources/inetutils-1.9.4# mv -v /usr/bin/ifconfig /sbin
renamed '/usr/bin/ifconfig' -> '/sbin/ifconfig'
```

清除软件包

```sh
(lfs chroot) root:/sources/inetutils-1.9.4# cd ..
(lfs chroot) root:/sources# rm -rf inetutils-1.9.4
```

### 8.40 安装 Perl-5.32.0

Perl 包含实用提取和报告语言。

解压软件包

```sh
(lfs chroot) root:/sources# tar xf perl-5.32.0.tar.xz 
(lfs chroot) root:/sources# cd perl-5.32.0
```

这个版本的 Perl 现在构建了 Compress::Raw::Zlib 和 Compress::Raw::BZip2 模块。默认情况下，Perl 将使用源的内部副本进行构建。发出以下命令，以便 Perl 使用系统上安装的库：

```sh
(lfs chroot) root:/sources/perl-5.32.0# export BUILD_ZLIB=False
(lfs chroot) root:/sources/perl-5.32.0# export BUILD_BZIP2=0
```

要完全控制 Perl 的设置方式，您可以从以下命令中删除 `-des` 选项并手动选择此包的构建方式。或者，使用完全如下的命令来使用 Perl 自动检测的默认值，同时编译并测试：

```sh
# 注意：下面的命令耗时非常长！
time { sh Configure -des                                  \
             -Dprefix=/usr                                \
             -Dvendorprefix=/usr                          \
             -Dprivlib=/usr/lib/perl5/5.32/core_perl      \
             -Darchlib=/usr/lib/perl5/5.32/core_perl      \
             -Dsitelib=/usr/lib/perl5/5.32/site_perl      \
             -Dsitearch=/usr/lib/perl5/5.32/site_perl     \
             -Dvendorlib=/usr/lib/perl5/5.32/vendor_perl  \
             -Dvendorarch=/usr/lib/perl5/5.32/vendor_perl \
             -Dman1dir=/usr/share/man/man1                \
             -Dman3dir=/usr/share/man/man3                \
             -Dpager="/usr/bin/less -isR"                 \
             -Duseshrplib                                 \
             -Dusethreads && make && make test; }

# 测试完成后输出的信息：
All tests successful.
Elapsed: 1253 sec
u=16.05  s=11.51  cu=771.78  cs=140.66  scripts=2554  tests=1220614

real    27m32.933s
user    18m56.738s
sys     3m16.717s
```

安装软件包并清理：

```sh
(lfs chroot) root:/sources/perl-5.32.0# make install
(lfs chroot) root:/sources/perl-5.32.0# unset BUILD_ZLIB BUILD_BZIP2
```

清理软件包

```sh
(lfs chroot) root:/sources/perl-5.32.0# cd ..
(lfs chroot) root:/sources# rm -rf perl-5.32.0
```

### 8.41 安装 XML::Parser-2.46

XML::Parser 模块是 James Clark 的 XML 解析器 Expat 的 Perl 接口。

解压软件包

```sh
(lfs chroot) root:/sources# tar xf XML-Parser-2.46.tar.gz 
(lfs chroot) root:/sources# cd XML-Parser-2.46
```

配置编译并测试：

```sh
time { perl Makefile.PL && make && make test; }

# 测试完成后输出信息：
All tests successful.
Files=15, Tests=140,  1 wallclock secs ( 0.07 usr  0.06 sys +  0.99 cusr  0.14 csys =  1.26 CPU)
Result: PASS

real    0m5.430s
user    0m4.764s
sys     0m0.599s
```

安装软件包

```sh
time { make install; }

# 安装完成后输出信息：
Installing /usr/share/man/man3/XML::Parser::Expat.3
Installing /usr/share/man/man3/XML::Parser::Style::Subs.3
Installing /usr/share/man/man3/XML::Parser::Style::Tree.3
Appending installation info to /usr/lib/perl5/5.32/core_perl/perllocal.pod

real    0m0.507s
user    0m0.374s
sys     0m0.100s
```

清除软件包

```sh
(lfs chroot) root:/sources/XML-Parser-2.46# cd ..
(lfs chroot) root:/sources# rm -rf XML-Parser-2.46
```

### 8.42 安装 Intltool-0.51.0

Intltool 是一种国际化工具，用于从源文件中提取可翻译的字符串。

解压软件包

```sh
(lfs chroot) root:/sources# tar xf intltool-0.51.0.tar.gz 
(lfs chroot) root:/sources# cd intltool-0.51.0
```

首先修复一个由 perl-5.22 及更高版本引起的警告：

```sh
(lfs chroot) root:/sources/intltool-0.51.0# sed -i 's:\\\${:\\\$\\{:' intltool-update.in
```

由于所有的反斜杠，上面的正则表达式看起来不寻常。它的作用是在序列 `'\${'` 中的右大括号字符之前添加一个反斜杠，从而得到 `'\$\{'`。

配置编译并测试：

```sh
time { ./configure --prefix=/usr && make && make check; }

# 测试完成后输出信息：
============================================================================
Testsuite summary for intltool 0.51.0
============================================================================
# TOTAL: 1
# PASS:  1
# SKIP:  0
# XFAIL: 0
# FAIL:  0
# XPASS: 0
# ERROR: 0
============================================================================
make[4]: Leaving directory '/sources/intltool-0.51.0/tests'
make[3]: Leaving directory '/sources/intltool-0.51.0/tests'
make[2]: Leaving directory '/sources/intltool-0.51.0/tests'
make[1]: Leaving directory '/sources/intltool-0.51.0/tests'
Making check in doc
make[1]: Entering directory '/sources/intltool-0.51.0/doc'
make[1]: Nothing to be done for 'check'.
make[1]: Leaving directory '/sources/intltool-0.51.0/doc'
make[1]: Entering directory '/sources/intltool-0.51.0'
make[1]: Nothing to be done for 'check-am'.
make[1]: Leaving directory '/sources/intltool-0.51.0'

real    0m4.296s
user    0m2.721s
sys     0m0.637s
```

安装软件包

```sh
time { make install && install -v -Dm644 doc/I18N-HOWTO /usr/share/doc/intltool-0.51.0/I18N-HOWTO; }

# 安装完成后输出信息：
make[2]: Leaving directory '/sources/intltool-0.51.0'
make[1]: Leaving directory '/sources/intltool-0.51.0'
install: creating directory '/usr/share/doc/intltool-0.51.0'
'doc/I18N-HOWTO' -> '/usr/share/doc/intltool-0.51.0/I18N-HOWTO'

real    0m0.163s
user    0m0.077s
sys     0m0.067s
```

清除软件包

```sh
(lfs chroot) root:/sources/intltool-0.51.0# cd ..
(lfs chroot) root:/sources# rm -rf intltool-0.51.0
```

### 8.43 安装 Autoconf-2.69

Autoconf 包含用于生成可自动配置源代码的 shell 脚本的程序。

解压软件包

```sh
(lfs chroot) root:/sources# tar xf autoconf-2.69.tar.xz 
(lfs chroot) root:/sources# cd autoconf-2.69
```

首先修复由 Perl 5.28 生成的错误。

```sh
(lfs chroot) root:/sources/autoconf-2.69# sed -i '361 s/{/\\{/' bin/autoscan.in
```

配置编译并测试：

```sh
time { ./configure --prefix=/usr && make && make check; }

# 测试完成后输出信息：
## ------------- ##
## Test results. ##
## ------------- ##

ERROR: 450 tests were run,
137 failed (4 expected failures).
53 tests were skipped.
## -------------------------- ##
## testsuite.log was created. ##
## -------------------------- ##

Please send `tests/testsuite.log' and all information you think might help:

   To: <bug-autoconf@gnu.org>
   Subject: [GNU Autoconf 2.69] testsuite: 77 232 233 270 272 273 274 276 277 281 282 287 288 289 290 291 292 293 294 295 296 297 298 299 300 301 302 303 348 349 353 354 355 356 357 358 362 363 364 365 366 367 368 369 370 371 372 374 380 381 382 383 384 385 386 387 388 389 390 391 404 405 406 407 418 419 420 421 422 423 424 425 426 427 431 432 433 434 435 436 437 438 443 444 445 446 447 448 449 450 451 452 453 454 455 456 457 458 459 460 461 462 463 471 472 473 474 475 476 477 478 479 480 481 482 483 484 485 486 487 488 489 490 491 492 493 494 495 496 497 498 499 501 failed

You may investigate any problem if you feel able to do so, in which
case the test suite provides a good starting point.  Its output may
be found below `tests/testsuite.dir'.

make[3]: *** [Makefile:596: check-local] Error 1
make[3]: Leaving directory '/sources/autoconf-2.69/tests'
make[2]: *** [Makefile:411: check-am] Error 2
make[2]: Leaving directory '/sources/autoconf-2.69/tests'
make[1]: *** [Makefile:357: check-recursive] Error 1
make[1]: Leaving directory '/sources/autoconf-2.69'
make: *** [Makefile:665: check] Error 2

real    12m22.859s
user    8m29.387s
sys     2m45.522s
```

可以看到，以上编译测试过程出现了较多错误。在手册中已指出 `The test suite is currently broken by bash-5 and libtool-2.4.3. `，这说明测试套件有可能本身就存在问题，因此，对于以上编译测试中出现的问题，我们暂时忽略。

安装软件包

```sh
time { make install; }

# 安装完成后输出信息：
make[3]: Leaving directory '/sources/autoconf-2.69/man'
make[2]: Leaving directory '/sources/autoconf-2.69/man'
make[1]: Leaving directory '/sources/autoconf-2.69'

real    0m0.757s
user    0m0.335s
sys     0m0.277s
```

清除软件包

```sh
(lfs chroot) root:/sources/autoconf-2.69# cd ..
(lfs chroot) root:/sources# rm -rf autoconf-2.69
```

### 8.44 安装 Automake-1.16.2

Automake 包含用于生成用于 Autoconf 的 Makefile 的程序。

解压软件包

```sh
(lfs chroot) root:/sources# tar xf automake-1.16.2.tar.xz 
(lfs chroot) root:/sources# cd automake-1.16.2
```

修复失败的测试：

```sh
(lfs chroot) root:/sources/automake-1.16.2# sed -i "s/''/etags/" t/tags-lisp-space.sh
```

配置编译并测试，由于个别测试的内部延迟，使用 -j4 make 选项可以加快测试速度，即使在只有一个处理器的系统上也是如此。：

```sh
time { ./configure --prefix=/usr --docdir=/usr/share/doc/automake-1.16.2 && make && make -j4 check; }

# 测试完成后输出信息：
============================================================================
Testsuite summary for GNU Automake 1.16.2
============================================================================
# TOTAL: 2915
# PASS:  2719
# SKIP:  157
# XFAIL: 39
# FAIL:  0
# XPASS: 0
# ERROR: 0
============================================================================
make[2]: Leaving directory '/sources/automake-1.16.2'
make[1]: Leaving directory '/sources/automake-1.16.2'

real    40m37.297s
user    67m38.496s
sys     32m31.625s
```

安装软件包

```sh
time { make install; }

# 安装完成后输出信息：
make  install-data-hook
make[2]: Entering directory '/sources/automake-1.16.2'
 chmod +x '/usr/share/automake-1.16/config.guess'
 chmod +x '/usr/share/automake-1.16/config.sub'
 chmod +x '/usr/share/automake-1.16/install-sh'
 chmod +x '/usr/share/automake-1.16/mdate-sh'
 chmod +x '/usr/share/automake-1.16/missing'
 chmod +x '/usr/share/automake-1.16/mkinstalldirs'
 chmod +x '/usr/share/automake-1.16/ylwrap'
 chmod +x '/usr/share/automake-1.16/depcomp'
 chmod +x '/usr/share/automake-1.16/compile'
 chmod +x '/usr/share/automake-1.16/py-compile'
 chmod +x '/usr/share/automake-1.16/ar-lib'
 chmod +x '/usr/share/automake-1.16/test-driver'
 chmod +x '/usr/share/automake-1.16/tap-driver.sh'
make[2]: Leaving directory '/sources/automake-1.16.2'
make[1]: Leaving directory '/sources/automake-1.16.2'

real    0m0.642s
user    0m0.352s
sys     0m0.226s
```

清除软件包

```sh
(lfs chroot) root:/sources/automake-1.16.2# cd ..
(lfs chroot) root:/sources# rm -rf automake-1.16.2
```

### 8.45 安装 Kmod-27

Kmod 包含用于加载内核模块的库和实用程序

解压软件包

```sh
(lfs chroot) root:/sources# tar xf kmod-27.tar.xz 
(lfs chroot) root:/sources# cd kmod-27
```

配置编译并安装：

```sh
time { ./configure --prefix=/usr   \
            --bindir=/bin          \
            --sysconfdir=/etc      \
            --with-rootlibdir=/lib \
            --with-xz              \
            --with-zlib && make && make install; }

# 测试完成后输出信息：
make[2]: Nothing to be done for 'install-exec-am'.
 /bin/mkdir -p '/usr/share/man/man5'
 /usr/bin/install -c -m 644 depmod.d.5 modprobe.d.5 modules.dep.5 modules.dep.bin.5 '/usr/share/man/man5'
 /bin/mkdir -p '/usr/share/man/man8'
 /usr/bin/install -c -m 644 kmod.8 depmod.8 insmod.8 lsmod.8 rmmod.8 modprobe.8 modinfo.8 '/usr/share/man/man8'

real    0m23.826s
user    0m16.846s
sys     0m5.751s
```

这个包没有附带可以在 LFS chroot 环境中运行的测试套件。需要最小的 git 程序，并且一些测试不会在 git 存储库之外运行。

创建符号链接以与 Module-Init-Tools（以前处理 Linux 内核模块的包）兼容：

```sh
# 执行脚本
for target in depmod insmod lsmod modinfo modprobe rmmod; do
  ln -sfv ../bin/kmod /sbin/$target
done
# 执行完毕后会输出以下信息：
'/sbin/depmod' -> '../bin/kmod'
'/sbin/insmod' -> '../bin/kmod'
'/sbin/lsmod' -> '../bin/kmod'
'/sbin/modinfo' -> '../bin/kmod'
'/sbin/modprobe' -> '../bin/kmod'
'/sbin/rmmod' -> '../bin/kmod'
# 创建链接
(lfs chroot) root:/sources/kmod-27# ln -sfv kmod /bin/lsmod
'/bin/lsmod' -> 'kmod'
```

清除软件包

```sh
(lfs chroot) root:/sources/kmod-27# cd ..
(lfs chroot) root:/sources# rm -rf kmod-27
```

### 8.46 安装 Libelf from Elfutils-0.180

Libelf 是一个用于处理 ELF（可执行和可链接格式）文件的库。

解压软件包

```sh
(lfs chroot) root:/sources# tar xf elfutils-0.180.tar.bz2 
(lfs chroot) root:/sources# cd elfutils-0.180
```

配置编译并检查：

```sh
time { ./configure --prefix=/usr --disable-debuginfod --libdir=/lib && make && make check; }

# 检查完成后输出信息：
============================================================================
Testsuite summary for elfutils 0.180
============================================================================
# TOTAL: 218
# PASS:  213
# SKIP:  5
# XFAIL: 0
# FAIL:  0
# XPASS: 0
# ERROR: 0
============================================================================

real    4m27.881s
user    3m13.298s
sys     0m52.794s
```

只安装 Libelf：

```sh
(lfs chroot) root:/sources/elfutils-0.180# make -C libelf install
(lfs chroot) root:/sources/elfutils-0.180# install -vm644 config/libelf.pc /usr/lib/pkgconfig
(lfs chroot) root:/sources/elfutils-0.180# rm /lib/libelf.a
```

清除软件包

```sh
(lfs chroot) root:/sources/elfutils-0.180# cd ..
(lfs chroot) root:/sources# rm -rf elfutils-0.180
```

### 8.47 安装 Libffi-3.3

Libffi 库为各种调用约定提供了可移植的高级编程接口。这允许程序员在运行时调用由调用接口描述指定的任何函数。

与 GMP 类似，libffi 使用特定于所用处理器的优化构建。如果为另一个系统构建，请导出 CFLAGS 和 CXXFLAGS 以指定架构的通用构建。如果不这样做，所有链接到 libffi 的应用程序都会触发非法操作错误。

解压软件包

```sh
(lfs chroot) root:/sources# tar xf libffi-3.3.tar.gz
(lfs chroot) root:/sources# cd libffi-3.3
```

配置编译并检查：

```sh
time { ./configure --prefix=/usr --disable-static --with-gcc-arch=native && make && make check; }

# 检查完成后输出信息：
                === libffi Summary ===

# of expected passes            2284
make[3]: Leaving directory '/sources/libffi-3.3/x86_64-pc-linux-gnu/testsuite'
make[2]: Leaving directory '/sources/libffi-3.3/x86_64-pc-linux-gnu/testsuite'
Making check in man
make[2]: Entering directory '/sources/libffi-3.3/x86_64-pc-linux-gnu/man'
make[2]: Nothing to be done for 'check'.
make[2]: Leaving directory '/sources/libffi-3.3/x86_64-pc-linux-gnu/man'
Making check in doc
make[2]: Entering directory '/sources/libffi-3.3/x86_64-pc-linux-gnu/doc'
make[2]: Nothing to be done for 'check'.
make[2]: Leaving directory '/sources/libffi-3.3/x86_64-pc-linux-gnu/doc'
make[2]: Entering directory '/sources/libffi-3.3/x86_64-pc-linux-gnu'
make[2]: Leaving directory '/sources/libffi-3.3/x86_64-pc-linux-gnu'
make[1]: Leaving directory '/sources/libffi-3.3/x86_64-pc-linux-gnu'

real    7m51.231s
user    6m59.217s
sys     0m44.474s
(lfs chroot) root:/sources/libffi-3.3# 
```

安装软件包

```sh
time { make install; }

# 安装完成后输出信息：
make[3]: Leaving directory '/sources/libffi-3.3/x86_64-pc-linux-gnu'
make[2]: Leaving directory '/sources/libffi-3.3/x86_64-pc-linux-gnu'
make[1]: Leaving directory '/sources/libffi-3.3/x86_64-pc-linux-gnu'

real    0m0.616s
user    0m0.364s
sys     0m0.151s
```

清除软件包

```sh
(lfs chroot) root:/sources/libffi-3.3# cd ..
(lfs chroot) root:/sources# rm -rf libffi-3.3
```

### 8.48 安装 OpenSSL-1.1.1g

OpenSSL 包含与加密相关的管理工具和库。这些对于为其他包提供加密功能非常有用，例如 OpenSSH、电子邮件应用程序和 Web 浏览器（用于访问 HTTPS 站点）。

解压软件包

```sh
(lfs chroot) root:/sources# tar xf openssl-1.1.1g.tar.gz 
(lfs chroot) root:/sources# cd openssl-1.1.1g
```

配置编译并检查：

```sh
time { ./config --prefix=/usr  \
         --openssldir=/etc/ssl \
         --libdir=lib          \
         shared                \
         zlib-dynamic && make && make test; }

# 检查完成后输出信息：
Result: PASS
make[1]: Leaving directory '/sources/openssl-1.1.1g'

real    7m52.965s
user    6m31.131s
sys     1m11.118s
```

已知一项测试 `30-test_afalg.t` 在某些内核配置上失败（它显然假设已选择了某些未指定的加密选项，我们可以忽略）。

安装软件包

```sh
time { sed -i '/INSTALL_LIBS/s/libcrypto.a libssl.a//' Makefile && make MANSUFFIX=ssl install; }

# 安装完成后输出信息：
/usr/share/doc/openssl/html/man7/X25519.html
/usr/share/doc/openssl/html/man7/X448.html -> /usr/share/doc/openssl/html/man7/X25519.html
/usr/share/doc/openssl/html/man7/x509.html

real    2m52.641s
user    2m28.528s
sys     0m19.168s
```

如果需要，请安装文档：

```sh
(lfs chroot) root:/sources/openssl-1.1.1g# mv -v /usr/share/doc/openssl /usr/share/doc/openssl-1.1.1g
renamed '/usr/share/doc/openssl' -> '/usr/share/doc/openssl-1.1.1g'
(lfs chroot) root:/sources/openssl-1.1.1g# cp -vfr doc/* /usr/share/doc/openssl-1.1.1g
```

清除软件包

```sh
(lfs chroot) root:/sources/openssl-1.1.1g# cd ..
(lfs chroot) root:/sources# rm -rf openssl-1.1.1g
```

### 8.49 安装 Python-3.8.5

Python 3 包含 Python 开发环境。它对于面向对象编程、编写脚本、大型程序原型或开发整个应用程序非常有用。

解压软件包

```sh
(lfs chroot) root:/sources# tar xf Python-3.8.5.tar.xz            
(lfs chroot) root:/sources# cd Python-3.8.5
```

配置编译并安装：

```sh
time { ./configure --prefix=/usr \
            --enable-shared      \
            --with-system-expat  \
            --with-system-ffi    \
            --with-ensurepip=yes && make && make install; }

# 检查完成后输出信息：
Processing /tmp/tmp74k0e_eh/setuptools-47.1.0-py3-none-any.whl
Processing /tmp/tmp74k0e_eh/pip-20.1.1-py2.py3-none-any.whl
Installing collected packages: setuptools, pip
Successfully installed pip-20.1.1 setuptools-47.1.0

real    6m10.517s
user    5m31.379s
sys     0m50.269s
```

要测试结果，可以执行 `make test`。一些需要网络连接或附加包的测试被跳过。名为 `test_normalization` 的测试会失败，因为尚未完成网络配置。为了获得更全面的结果，可以在 BLFS 中重新安装 Python3 时重新运行测试。(我不运行此次测试程序)

更改权限并创建链接

```sh
(lfs chroot) root:/sources/Python-3.8.5# chmod -v 755 /usr/lib/libpython3.8.so
mode of '/usr/lib/libpython3.8.so' retained as 0755 (rwxr-xr-x)
(lfs chroot) root:/sources/Python-3.8.5# chmod -v 755 /usr/lib/libpython3.so
mode of '/usr/lib/libpython3.so' retained as 0755 (rwxr-xr-x)
(lfs chroot) root:/sources/Python-3.8.5# ln -sfv pip3.8 /usr/bin/pip3
'/usr/bin/pip3' -> 'pip3.8'
```

如果需要，安装预先格式化的文档：

```sh
(lfs chroot) root:/sources/Python-3.8.5# install -v -dm755 /usr/share/doc/python-3.8.5/html
install: creating directory '/usr/share/doc/python-3.8.5'
install: creating directory '/usr/share/doc/python-3.8.5/html'
# 继续执行以下命令
tar --strip-components=1  \
    --no-same-owner       \
    --no-same-permissions \
    -C /usr/share/doc/python-3.8.5/html \
    -xvf ../python-3.8.5-docs-html.tar.bz2
```

清除软件包

```sh
(lfs chroot) root:/sources/Python-3.8.5# cd ..
(lfs chroot) root:/sources# rm -rf Python-3.8.5
```

### 8.50 安装 Ninja-1.10.0

Ninja 是一个专注于速度的小型构建系统。

Tips: 如果不使用 systemd，则 LFS 不严格要求此部分。然而，Ninja 和 Meson(下文会构建) 进行了强大的构建系统整合，预计会越来越频繁地使用。 BLFS 书中的几个包都需要它。

解压软件包

```sh
(lfs chroot) root:/sources# tar xf ninja-1.10.0.tar.gz 
(lfs chroot) root:/sources# cd ninja-1.10.0
```

运行时，ninja 通常并行运行最大数量的进程。默认情况下，这是系统上的内核数加 2。在某些情况下，这可能会使 CPU 过热或使系统内存不足。如果从命令行运行，传递 -jN 参数会限制并行进程的数量，但有些包嵌入了 ninja 的执行，并没有传递 -j 参数。

使用下面的可选过程允许用户通过环境变量 NINJAJOBS 限制并行进程的数量：

```sh
(lfs chroot) root:/sources/ninja-1.10.0# export NINJAJOBS=4
```

以上命令将 ninja 限制为四个并行进程。

如果需要，通过运行以下命令添加使用环境变量 NINJAJOBS 的功能：

```sh
# 执行以下脚本
sed -i '/int Guess/a \
  int   j = 0;\
  char* jobs = getenv( "NINJAJOBS" );\
  if ( jobs != NULL ) j = atoi( jobs );\
  if ( j > 0 ) return j;\
' src/ninja.cc
```

使用以下命令构建 Ninja：

```sh
(lfs chroot) root:/sources/ninja-1.10.0# python3 configure.py --bootstrap
# 输出以下内容：
bootstrapping ninja...
warning: A compatible version of re2c (>= 0.11.3) was not found; changes to src/*.in.cc will not affect your build.
wrote build.ninja.
bootstrap complete.  rebuilding...
[29/29] LINK ninja
```

要测试结果，请执行：

```sh
(lfs chroot) root:/sources/ninja-1.10.0# ./ninja ninja_test
# 输出内容：
[19/19] LINK ninja_test
(lfs chroot) root:/sources/ninja-1.10.0# ./ninja_test --gtest_filter=-SubprocessTest.SetWithLots
# 输出内容：
[341/341] ElideMiddle.ElideInTheMiddle
passed
```

安装软件包

```sh
(lfs chroot) root:/sources/ninja-1.10.0# install -vm755 ninja /usr/bin/
'ninja' -> '/usr/bin/ninja'
(lfs chroot) root:/sources/ninja-1.10.0# install -vDm644 misc/bash-completion /usr/share/bash-completion/completions/ninja
'misc/bash-completion' -> '/usr/share/bash-completion/completions/ninja'
(lfs chroot) root:/sources/ninja-1.10.0# install -vDm644 misc/zsh-completion /usr/share/zsh/site-functions/_ninja
install: creating directory '/usr/share/zsh'
install: creating directory '/usr/share/zsh/site-functions'
'misc/zsh-completion' -> '/usr/share/zsh/site-functions/_ninja'
```

清除软件包

```sh
(lfs chroot) root:/sources/ninja-1.10.0# cd ..
(lfs chroot) root:/sources# rm -rf ninja-1.10.0
```

### 8.51 安装 Meson-0.55.0

Meson 是一个开源构建系统，旨在速度极快且用户友好。

Tips: 如果不使用 systemd，则 LFS 不严格要求此部分。然而，Ninja(上文已构建) 和 Meson 进行了强大的构建系统整合，预计会越来越频繁地使用。 BLFS 书中的几个包都需要它。

解压软件包

```sh
(lfs chroot) root:/sources# tar xf meson-0.55.0.tar.gz 
(lfs chroot) root:/sources# cd meson-0.55.0
```

使用以下命令编译 Meson：

```sh
(lfs chroot) root:/sources/meson-0.55.0# python3 setup.py build

# 编译完成后输出：
creating build/lib/mesonbuild/dependencies/data
copying mesonbuild/dependencies/data/CMakeLists.txt -> build/lib/mesonbuild/dependencies/data
copying mesonbuild/dependencies/data/CMakeListsLLVM.txt -> build/lib/mesonbuild/dependencies/data
copying mesonbuild/dependencies/data/CMakePathInfo.txt -> build/lib/mesonbuild/dependencies/data
```

此软件包不附带测试套件。

安装软件包

```sh
(lfs chroot) root:/sources/meson-0.55.0# python3 setup.py install --root=dest
# 安装完成后输出：
writing manifest file 'meson.egg-info/SOURCES.txt'
Copying meson.egg-info to dest/usr/lib/python3.8/site-packages/meson-0.55.0-py3.8.egg-info
running install_scripts
Installing meson script to dest/usr/bin

(lfs chroot) root:/sources/meson-0.55.0# cp -rv dest/* /
# 完成后输出：
'dest/usr/share/polkit-1' -> '/usr/share/polkit-1'
'dest/usr/share/polkit-1/actions' -> '/usr/share/polkit-1/actions'
'dest/usr/share/polkit-1/actions/com.mesonbuild.install.policy' -> '/usr/share/polkit-1/actions/com.mesonbuild.install.policy'
'dest/usr/bin/meson' -> '/usr/bin/meson'
```

清理软件包

```sh
(lfs chroot) root:/sources/meson-0.55.0# cd ..
(lfs chroot) root:/sources# rm -rf meson-0.55.0
```

### 8.52 安装 Coreutils-8.32

Coreutils 包含用于显示和设置基本系统特征的实用程序。

解压软件包

```sh
(lfs chroot) root:/sources# tar xf coreutils-8.32.tar.xz 
(lfs chroot) root:/sources# cd coreutils-8.32
```

POSIX 要求来自 Coreutils 的程序即使在多字节语言环境中也能正确识别字符边界。以下补丁修复了这种不合规性和其他与国际化相关的错误。

```sh
(lfs chroot) root:/sources/coreutils-8.32# patch -Np1 -i ../coreutils-8.32-i18n-1.patch
patching file bootstrap.conf
patching file configure.ac
patching file lib/linebuffer.h
patching file lib/mbfile.c
patching file lib/mbfile.h
patching file m4/mbfile.m4
patching file src/cut.c
patching file src/expand.c
patching file src/expand-common.c
patching file src/expand-common.h
patching file src/fold.c
patching file src/join.c
patching file src/pr.c
patching file src/sort.c
patching file src/unexpand.c
patching file src/uniq.c
patching file tests/expand/mb.sh
patching file tests/i18n/sort.sh
patching file tests/local.mk
patching file tests/misc/expand.pl
patching file tests/misc/fold.pl
patching file tests/misc/join.pl
patching file tests/misc/sort-mb-tests.sh
patching file tests/misc/sort-merge.pl
patching file tests/misc/sort.pl
patching file tests/misc/unexpand.pl
patching file tests/misc/uniq.pl
patching file tests/pr/pr-tests.pl
patching file tests/unexpand/mb.sh
```

注意：以前在此补丁中发现了许多错误。向 Coreutils 维护者报告新错误时，请先检查它们是否可以在没有此补丁的情况下重现。

禁止在某些机器上可以永远循环的测试：

```sh
(lfs chroot) root:/sources/coreutils-8.32# sed -i '/test.lock/s/^/#/' gnulib-tests/gnulib.mk
```

配置并编译

```sh
time { autoreconf -fiv && FORCE_UNSAFE_CONFIGURE=1 ./configure \
            --prefix=/usr            \
            --enable-no-install-program=kill,uptime && make; }

# 编译完成后输出以下信息：
make[4]: Leaving directory '/sources/coreutils-8.32/gnulib-tests'
make[3]: Leaving directory '/sources/coreutils-8.32/gnulib-tests'
make[2]: Leaving directory '/sources/coreutils-8.32/gnulib-tests'
make[1]: Leaving directory '/sources/coreutils-8.32'

real    4m19.931s
user    3m1.109s
sys     0m49.541s
```

现在测试套件已准备好运行。首先，运行以 root 用户身份运行的测试：

```sh
(lfs chroot) root:/sources/coreutils-8.32# make NON_ROOT_USERNAME=tester check-root

# 完成后输出：
============================================================================
Testsuite summary for GNU coreutils 8.32
============================================================================
# TOTAL: 32
# PASS:  19
# SKIP:  13
# XFAIL: 0
# FAIL:  0
# XPASS: 0
# ERROR: 0
============================================================================
make[5]: Leaving directory '/sources/coreutils-8.32'
  GEN      check-README
  GEN      check-duplicate-no-install
  GEN      sc-avoid-builtin
  GEN      sc-avoid-io
  GEN      sc-avoid-non-zero
  GEN      sc-avoid-path
  GEN      sc-avoid-timezone
  GEN      sc-avoid-zeroes
  GEN      sc-exponent-grouping
  GEN      sc-lower-case-var
  GEN      check-texinfo
make[4]: Leaving directory '/sources/coreutils-8.32'
make[3]: Leaving directory '/sources/coreutils-8.32'
make[2]: Leaving directory '/sources/coreutils-8.32'
make[1]: Leaving directory '/sources/coreutils-8.32'
```

接下来将切换至 tester 用户运行其余的测试。某些测试要求用户是多个组的成员。为了不跳过这些测试，添加一个临时组并使用户 tester 成为其中的一部分：

```sh
(lfs chroot) root:/sources/coreutils-8.32# echo "dummy:x:102:tester" >> /etc/group
```

修复一些权限，以便非 root 用户可以编译和运行测试：

```sh
(lfs chroot) root:/sources/coreutils-8.32# chown -Rv tester .
```

现在运行测试：

```sh
(lfs chroot) root:/sources/coreutils-8.32# su tester -c "PATH=$PATH make RUN_EXPENSIVE_TESTS=yes check"

# 完成后输出：
============================================================================
Testsuite summary for GNU coreutils 8.32
============================================================================
# TOTAL: 345
# PASS:  328
# SKIP:  17
# XFAIL: 0
# FAIL:  0
# XPASS: 0
# ERROR: 0
============================================================================
make[6]: Leaving directory '/sources/coreutils-8.32/gnulib-tests'
make[5]: Leaving directory '/sources/coreutils-8.32/gnulib-tests'
make[4]: Leaving directory '/sources/coreutils-8.32/gnulib-tests'
make[3]: Leaving directory '/sources/coreutils-8.32/gnulib-tests'
make[2]: Leaving directory '/sources/coreutils-8.32/gnulib-tests'
make[1]: Leaving directory '/sources/coreutils-8.32'
```

已知 `test-getlogin` 测试在 LFS chroot 环境中失败。

删除临时组：

```sh
(lfs chroot) root:/sources/coreutils-8.32# sed -i '/dummy/d' /etc/group
```

安装软件包：

```sh
make install

# 安装完成后输出：
Making install in .
make[4]: Entering directory '/sources/coreutils-8.32/gnulib-tests'
make[5]: Entering directory '/sources/coreutils-8.32/gnulib-tests'
make[5]: Leaving directory '/sources/coreutils-8.32/gnulib-tests'
make[4]: Leaving directory '/sources/coreutils-8.32/gnulib-tests'
make[3]: Leaving directory '/sources/coreutils-8.32/gnulib-tests'
make[2]: Leaving directory '/sources/coreutils-8.32/gnulib-tests'
make[1]: Leaving directory '/sources/coreutils-8.32'
```

将程序移动到 FHS 指定的位置：

```sh
(lfs chroot) root:/sources/coreutils-8.32# mv -v /usr/bin/{cat,chgrp,chmod,chown,cp,date,dd,df,echo} /bin
renamed '/usr/bin/cat' -> '/bin/cat'
renamed '/usr/bin/chgrp' -> '/bin/chgrp'
renamed '/usr/bin/chmod' -> '/bin/chmod'
renamed '/usr/bin/chown' -> '/bin/chown'
renamed '/usr/bin/cp' -> '/bin/cp'
renamed '/usr/bin/date' -> '/bin/date'
renamed '/usr/bin/dd' -> '/bin/dd'
renamed '/usr/bin/df' -> '/bin/df'
renamed '/usr/bin/echo' -> '/bin/echo'
(lfs chroot) root:/sources/coreutils-8.32# mv -v /usr/bin/{false,ln,ls,mkdir,mknod,mv,pwd,rm} /bin
renamed '/usr/bin/false' -> '/bin/false'
renamed '/usr/bin/ln' -> '/bin/ln'
renamed '/usr/bin/ls' -> '/bin/ls'
renamed '/usr/bin/mkdir' -> '/bin/mkdir'
renamed '/usr/bin/mknod' -> '/bin/mknod'
renamed '/usr/bin/mv' -> '/bin/mv'
renamed '/usr/bin/pwd' -> '/bin/pwd'
renamed '/usr/bin/rm' -> '/bin/rm'
(lfs chroot) root:/sources/coreutils-8.32# mv -v /usr/bin/{rmdir,stty,sync,true,uname} /bin
renamed '/usr/bin/rmdir' -> '/bin/rmdir'
renamed '/usr/bin/stty' -> '/bin/stty'
renamed '/usr/bin/sync' -> '/bin/sync'
renamed '/usr/bin/true' -> '/bin/true'
renamed '/usr/bin/uname' -> '/bin/uname'
(lfs chroot) root:/sources/coreutils-8.32# mv -v /usr/bin/chroot /usr/sbin
renamed '/usr/bin/chroot' -> '/usr/sbin/chroot'
(lfs chroot) root:/sources/coreutils-8.32# mv -v /usr/share/man/man1/chroot.1 /usr/share/man/man8/chroot.8
renamed '/usr/share/man/man1/chroot.1' -> '/usr/share/man/man8/chroot.8'
(lfs chroot) root:/sources/coreutils-8.32# sed -i 's/"1"/"8"/' /usr/share/man/man8/chroot.8
```

LFS-Bootscripts 包中的一些脚本依赖于 head、nice、sleep 和 touch。由于 /usr 在引导的早期和后期阶段可能不可用，这些二进制文件需要移动到根分区上以保持 FHS 合规性：

```sh
(lfs chroot) root:/sources/coreutils-8.32# mv -v /usr/bin/{head,nice,sleep,touch} /bin
renamed '/usr/bin/head' -> '/bin/head'
renamed '/usr/bin/nice' -> '/bin/nice'
renamed '/usr/bin/sleep' -> '/bin/sleep'
renamed '/usr/bin/touch' -> '/bin/touch'
```

清理软件包

```sh
(lfs chroot) root:/sources/coreutils-8.32# cd ..
(lfs chroot) root:/sources# rm -rf coreutils-8.32
```

### 8.53 安装 Check-0.15.2

Check 是 C 的单元测试框架

解压软件包

```sh
(lfs chroot) root:/sources# tar xf check-0.15.2.tar.gz 
(lfs chroot) root:/sources# cd check-0.15.2
```

配置并编译检查：

```sh
time { ./configure --prefix=/usr --disable-static && make && make check; }

# 检查完成后显示内容如下：
============================================================================
Testsuite summary for Check 0.15.2
============================================================================
# TOTAL: 9
# PASS:  9
# SKIP:  0
# XFAIL: 0
# FAIL:  0
# XPASS: 0
# ERROR: 0
============================================================================
make[3]: Leaving directory '/sources/check-0.15.2/tests'
make[2]: Leaving directory '/sources/check-0.15.2/tests'
make[1]: Leaving directory '/sources/check-0.15.2/tests'

real    6m24.650s
user    0m23.196s
sys     0m7.285s
```

安装软件包

```sh
time { make docdir=/usr/share/doc/check-0.15.2 install; }

# 安装完成后显示内容如下：
Making install in tests
make[1]: Entering directory '/sources/check-0.15.2/tests'
make[2]: Entering directory '/sources/check-0.15.2/tests'
make[2]: Nothing to be done for 'install-exec-am'.
make[2]: Nothing to be done for 'install-data-am'.
make[2]: Leaving directory '/sources/check-0.15.2/tests'
make[1]: Leaving directory '/sources/check-0.15.2/tests'

real    0m0.554s
user    0m0.348s
sys     0m0.179s
```

清理软件包

```sh
(lfs chroot) root:/sources/check-0.15.2# cd ..
(lfs chroot) root:/sources# rm -rf check-0.15.2
```

### 8.54 安装 Diffutils-3.7

Diffutils 包含显示文件或目录之间差异的程序。

解压软件包

```sh
(lfs chroot) root:/sources# tar xf diffutils-3.7.tar.xz 
(lfs chroot) root:/sources# cd diffutils-3.7
```

配置并编译检查：

```sh
time { ./configure --prefix=/usr && make && make check; }

# 检查完成后显示内容如下：
============================================================================
Testsuite summary for GNU diffutils 3.7
============================================================================
# TOTAL: 173
# PASS:  160
# SKIP:  13
# XFAIL: 0
# FAIL:  0
# XPASS: 0
# ERROR: 0
============================================================================
make[5]: Leaving directory '/sources/diffutils-3.7/gnulib-tests'
make[4]: Leaving directory '/sources/diffutils-3.7/gnulib-tests'
make[3]: Leaving directory '/sources/diffutils-3.7/gnulib-tests'
make[2]: Leaving directory '/sources/diffutils-3.7/gnulib-tests'
make[1]: Leaving directory '/sources/diffutils-3.7/gnulib-tests'
make[1]: Entering directory '/sources/diffutils-3.7'
make[1]: Nothing to be done for 'check-am'.
make[1]: Leaving directory '/sources/diffutils-3.7'
  GEN      public-submodule-commit

real    1m36.742s
user    0m56.432s
sys     0m18.539s
```

安装软件包

```sh
time { make install; }

# 安装完成后显示内容如下：
make[2]: Leaving directory '/sources/diffutils-3.7/gnulib-tests'
make[1]: Leaving directory '/sources/diffutils-3.7/gnulib-tests'
make[1]: Entering directory '/sources/diffutils-3.7'
make[2]: Entering directory '/sources/diffutils-3.7'
make[2]: Nothing to be done for 'install-exec-am'.
make[2]: Nothing to be done for 'install-data-am'.
make[2]: Leaving directory '/sources/diffutils-3.7'
make[1]: Leaving directory '/sources/diffutils-3.7'

real    0m1.356s
user    0m0.913s
sys     0m0.315s
```

清理软件包

```sh
(lfs chroot) root:/sources/diffutils-3.7# cd ..
(lfs chroot) root:/sources# rm -rf diffutils-3.7
```

### 8.55 安装 Gawk-5.1.0

Gawk 包含用于处理文本文件的程序。

解压软件包

```sh
(lfs chroot) root:/sources# tar xf gawk-5.1.0.tar.xz 
(lfs chroot) root:/sources# cd gawk-5.1.0
```

确保没有安装一些不需要的文件：

```sh
(lfs chroot) root:/sources/gawk-5.1.0# sed -i 's/extras//' Makefile.in
```

配置并编译检查：

```sh
time { ./configure --prefix=/usr && make && make check; }

# 检查完成后显示内容如下：
make[3]: Leaving directory '/sources/gawk-5.1.0/test'
======== Done with MPFR tests ========
make[3]: Entering directory '/sources/gawk-5.1.0/test'
ALL TESTS PASSED
make[3]: Leaving directory '/sources/gawk-5.1.0/test'
make[2]: Leaving directory '/sources/gawk-5.1.0/test'
make[1]: Leaving directory '/sources/gawk-5.1.0'

real    1m31.024s
user    0m56.196s
sys     0m15.171s
```

安装软件包

```sh
time { make install; }

# 安装完成后显示内容如下：
Making install in test
make[2]: Entering directory '/sources/gawk-5.1.0/test'
make[3]: Entering directory '/sources/gawk-5.1.0/test'
make[3]: Nothing to be done for 'install-exec-am'.
make[3]: Nothing to be done for 'install-data-am'.
make[3]: Leaving directory '/sources/gawk-5.1.0/test'
make[2]: Leaving directory '/sources/gawk-5.1.0/test'
make[1]: Leaving directory '/sources/gawk-5.1.0'

real    0m3.153s
user    0m1.579s
sys     0m1.000s
```

如果需要，请安装文档：

```sh
(lfs chroot) root:/sources/gawk-5.1.0# mkdir -v /usr/share/doc/gawk-5.1.0
mkdir: created directory '/usr/share/doc/gawk-5.1.0'
(lfs chroot) root:/sources/gawk-5.1.0# cp -v doc/{awkforai.txt,*.{eps,pdf,jpg}} /usr/share/doc/gawk-5.1.0
'doc/awkforai.txt' -> '/usr/share/doc/gawk-5.1.0/awkforai.txt'
'doc/api-figure1.eps' -> '/usr/share/doc/gawk-5.1.0/api-figure1.eps'
'doc/api-figure2.eps' -> '/usr/share/doc/gawk-5.1.0/api-figure2.eps'
'doc/api-figure3.eps' -> '/usr/share/doc/gawk-5.1.0/api-figure3.eps'
'doc/array-elements.eps' -> '/usr/share/doc/gawk-5.1.0/array-elements.eps'
'doc/general-program.eps' -> '/usr/share/doc/gawk-5.1.0/general-program.eps'
'doc/lflashlight.eps' -> '/usr/share/doc/gawk-5.1.0/lflashlight.eps'
'doc/process-flow.eps' -> '/usr/share/doc/gawk-5.1.0/process-flow.eps'
'doc/rflashlight.eps' -> '/usr/share/doc/gawk-5.1.0/rflashlight.eps'
'doc/statist.eps' -> '/usr/share/doc/gawk-5.1.0/statist.eps'
'doc/api-figure1.pdf' -> '/usr/share/doc/gawk-5.1.0/api-figure1.pdf'
'doc/api-figure2.pdf' -> '/usr/share/doc/gawk-5.1.0/api-figure2.pdf'
'doc/api-figure3.pdf' -> '/usr/share/doc/gawk-5.1.0/api-figure3.pdf'
'doc/array-elements.pdf' -> '/usr/share/doc/gawk-5.1.0/array-elements.pdf'
'doc/general-program.pdf' -> '/usr/share/doc/gawk-5.1.0/general-program.pdf'
'doc/lflashlight.pdf' -> '/usr/share/doc/gawk-5.1.0/lflashlight.pdf'
'doc/process-flow.pdf' -> '/usr/share/doc/gawk-5.1.0/process-flow.pdf'
'doc/rflashlight.pdf' -> '/usr/share/doc/gawk-5.1.0/rflashlight.pdf'
'doc/statist.pdf' -> '/usr/share/doc/gawk-5.1.0/statist.pdf'
'doc/statist.jpg' -> '/usr/share/doc/gawk-5.1.0/statist.jpg'
```

清理软件包

```sh
(lfs chroot) root:/sources/gawk-5.1.0# cd ..
(lfs chroot) root:/sources# rm -rf gawk-5.1.0
```

### 8.56 安装 Findutils-4.7.0

Findutils 包含查找文件的程序。这些程序用于递归搜索目录树以及创建、维护和搜索数据库（通常比递归查找快，但如果数据库最近没有更新则不可靠）。

解压软件包

```sh
(lfs chroot) root:/sources# tar xf findutils-4.7.0.tar.xz 
(lfs chroot) root:/sources# cd findutils-4.7.0
```

配置并编译：

```sh
time { ./configure --prefix=/usr --localstatedir=/var/lib/locate && make; }

# 编译完成后显示内容如下：
make[4]: Leaving directory '/sources/findutils-4.7.0/gnulib-tests'
make[3]: Leaving directory '/sources/findutils-4.7.0/gnulib-tests'
make[2]: Leaving directory '/sources/findutils-4.7.0/gnulib-tests'
make[2]: Entering directory '/sources/findutils-4.7.0'
make[2]: Leaving directory '/sources/findutils-4.7.0'
make[1]: Leaving directory '/sources/findutils-4.7.0'

real    1m13.384s
user    0m42.437s
sys     0m19.278s
```

要测试结果，请执行：

```sh
(lfs chroot) root:/sources/findutils-4.7.0# chown -Rv tester .
(lfs chroot) root:/sources/findutils-4.7.0# su tester -c "PATH=$PATH make check"

# 测试完成后显示内容：
============================================================================
Testsuite summary for GNU findutils 4.7.0
============================================================================
# TOTAL: 12
# PASS:  11
# SKIP:  1
# XFAIL: 0
# FAIL:  0
# XPASS: 0
# ERROR: 0
============================================================================
make[3]: Leaving directory '/sources/findutils-4.7.0'
make[2]: Leaving directory '/sources/findutils-4.7.0'
make[1]: Leaving directory '/sources/findutils-4.7.0'
  GEN      public-submodule-commit
```

安装软件包

```sh
time { make install; }

# 安装完成后显示内容如下：
make[2]: Nothing to be done for 'install-exec-am'.
make[2]: Nothing to be done for 'install-data-am'.
make[2]: Leaving directory '/sources/findutils-4.7.0'
make[1]: Leaving directory '/sources/findutils-4.7.0'

real    0m2.312s
user    0m1.428s
sys     0m0.658s
```

LFS-Bootscripts 包中的一些脚本依赖于 find。由于 /usr 在引导的早期阶段可能不可用，因此该程序需要位于根分区上。还需要修改 updatedb 脚本以更正显式路径：

```sh
(lfs chroot) root:/sources/findutils-4.7.0# mv -v /usr/bin/find /bin
renamed '/usr/bin/find' -> '/bin/find'
(lfs chroot) root:/sources/findutils-4.7.0# sed -i 's|find:=${BINDIR}|find:=/bin|' /usr/bin/updatedb
```

清理软件包

```sh
(lfs chroot) root:/sources/findutils-4.7.0# cd ..
(lfs chroot) root:/sources# rm -rf findutils-4.7.0
```

### 8.57 安装 Groff-1.22.4

Groff 包含用于处理和格式化文本的程序。

解压软件包

```sh
(lfs chroot) root:/sources# tar xf groff-1.22.4.tar.gz 
(lfs chroot) root:/sources# cd groff-1.22.4
```

Groff 期望环境变量 PAGE 包含默认的纸张大小。对于美国用户，PAGE=letter 比较合适。在其他地方，PAGE=A4 可能更合适。虽然在编译期间配置了默认纸张大小，但稍后可以通过将“A4”或“letter”回显到 `/etc/papersize` 文件来覆盖它。

配置并编译，此包不支持并行构建，因此使用了 `make -j1`：

```sh
# 请注意这里我将默认纸张大小设置为 A4
time { PAGE=A4 ./configure --prefix=/usr && make -j1 && make install; }

# 编译完成后显示内容如下：
make[3]: Leaving directory '/sources/groff-1.22.4'
make[2]: Leaving directory '/sources/groff-1.22.4'
make[1]: Leaving directory '/sources/groff-1.22.4'

real    1m34.012s
user    1m15.502s
sys     0m14.755s
```

清理软件包

```sh
(lfs chroot) root:/sources/groff-1.22.4# cd ..
(lfs chroot) root:/sources# rm -rf groff-1.22.4
```

### 8.58 安装 GRUB-2.04

GRUB 包含 GRand Unified Bootloader。

解压软件包

```sh
(lfs chroot) root:/sources# tar xf grub-2.04.tar.xz 
(lfs chroot) root:/sources# cd grub-2.04
```

配置编译并安装：

```sh
time { ./configure --prefix=/usr   \
            --sbindir=/sbin        \
            --sysconfdir=/etc      \
            --disable-efiemu       \
            --disable-werror && make && make install; }

# 编译完成后显示内容如下：
make[3]: Leaving directory '/sources/grub-2.04/util/bash-completion.d'
make[2]: Leaving directory '/sources/grub-2.04/util/bash-completion.d'
make[1]: Leaving directory '/sources/grub-2.04'

real    3m49.738s
user    2m41.081s
sys     0m46.802s
```

此软件包不附带测试套件。

移动文件

```sh
(lfs chroot) root:/sources/grub-2.04# mv -v /etc/bash_completion.d/grub /usr/share/bash-completion/completions
renamed '/etc/bash_completion.d/grub' -> '/usr/share/bash-completion/completions/grub'
```

使用 GRUB 使您的 LFS 系统可启动将在第 10.4 节“使用 GRUB 设置启动过程”中讨论。

清理软件包

```sh
(lfs chroot) root:/sources/grub-2.04# cd ..
(lfs chroot) root:/sources# rm -rf grub-2.04
```

### 8.59 安装 Less-551

Less 包含一个文本文件查看器。

解压软件包

```sh
(lfs chroot) root:/sources# tar xf less-551.tar.gz 
(lfs chroot) root:/sources# cd less-551
```

配置编译并安装：

```sh
time { ./configure --prefix=/usr --sysconfdir=/etc && make && make install; }

# 安装完成后显示内容如下：
./mkinstalldirs /usr/bin /usr/share/man/man1
/usr/bin/install -c less /usr/bin/less
/usr/bin/install -c lesskey /usr/bin/lesskey
/usr/bin/install -c lessecho /usr/bin/lessecho
/usr/bin/install -c -m 644 ./less.nro /usr/share/man/man1/less.1
/usr/bin/install -c -m 644 ./lesskey.nro /usr/share/man/man1/lesskey.1
/usr/bin/install -c -m 644 ./lessecho.nro /usr/share/man/man1/lessecho.1

real    0m16.382s
user    0m11.304s
sys     0m3.444s
```

清理软件包

```sh
(lfs chroot) root:/sources/less-551# cd ..
(lfs chroot) root:/sources# rm -rf less-551
```

### 8.60 安装 Gzip-1.10

Gzip 包含用于压缩和解压缩文件的程序。

解压软件包

```sh
(lfs chroot) root:/sources# tar xf gzip-1.10.tar.xz 
(lfs chroot) root:/sources# cd gzip-1.10
```

配置编译并检查：

```sh
time { ./configure --prefix=/usr && make && make check; }

# 检查完成后显示内容如下：
============================================================================
Testsuite summary for gzip 1.10
============================================================================
# TOTAL: 22
# PASS:  22
# SKIP:  0
# XFAIL: 0
# FAIL:  0
# XPASS: 0
# ERROR: 0
============================================================================
make[4]: Leaving directory '/sources/gzip-1.10/tests'
make[3]: Leaving directory '/sources/gzip-1.10/tests'
make[2]: Leaving directory '/sources/gzip-1.10/tests'
make[1]: Leaving directory '/sources/gzip-1.10'

real    0m39.525s
user    0m23.829s
sys     0m9.949s
```

安装软件包

```sh
time { make install; }

# 安装完成后显示内容如下：
Making install in tests
make[2]: Entering directory '/sources/gzip-1.10/tests'
make[3]: Entering directory '/sources/gzip-1.10/tests'
make[3]: Nothing to be done for 'install-exec-am'.
make[3]: Nothing to be done for 'install-data-am'.
make[3]: Leaving directory '/sources/gzip-1.10/tests'
make[2]: Leaving directory '/sources/gzip-1.10/tests'
make[1]: Leaving directory '/sources/gzip-1.10'

real    0m0.833s
user    0m0.543s
sys     0m0.210s
```

移动一个需要在根文件系统上的程序：

```sh
(lfs chroot) root:/sources/gzip-1.10# mv -v /usr/bin/gzip /bin
renamed '/usr/bin/gzip' -> '/bin/gzip'
```

清理软件包

```sh
(lfs chroot) root:/sources/gzip-1.10# cd ..
(lfs chroot) root:/sources# rm -rf gzip-1.10
```

### 8.61 安装 IPRoute2-5.8.0

IPRoute2 包含用于基本和高级基于 IPV4 的网络的程序。

解压软件包

```sh
(lfs chroot) root:/sources# tar xf iproute2-5.8.0.tar.xz 
(lfs chroot) root:/sources# cd iproute2-5.8.0
```

该包中包含的 arpd 程序将不会构建，因为它依赖于未安装在 LFS 中的 Berkeley DB。但是，仍会安装 arpd 目录和手册页。通过运行以下命令来防止这种情况。如果需要 arpd 二进制文件，可以在 http://www.linuxfromscratch.org/blfs/view/10.0/server/db.html 的 BLFS Book 中找到编译 Berkeley DB 的说明。

```sh
# 执行以下命令
(lfs chroot) root:/sources/iproute2-5.8.0# sed -i /ARPD/d Makefile
(lfs chroot) root:/sources/iproute2-5.8.0# rm -fv man/man8/arpd.8
removed 'man/man8/arpd.8'
```

还需要禁用构建两个需要 http://www.linuxfromscratch.org/blfs/view/10.0/postlfs/iptables.html 的模块。

```sh
(lfs chroot) root:/sources/iproute2-5.8.0# sed -i 's/.m_ipt.o//' tc/Makefile
```

编译并安装软件：

```sh
time { make && make DOCDIR=/usr/share/doc/iproute2-5.8.0 install; }

# 安装完成后显示内容如下：
install -m 0755 -d /usr/share/bash-completion/completions
install -m 0644 bash-completion/tc /usr/share/bash-completion/completions
install -m 0644 bash-completion/devlink /usr/share/bash-completion/completions
install -m 0644 include/bpf_elf.h /usr/include/iproute2

real    0m46.123s
user    0m39.774s
sys     0m5.489s
```

清理软件包

```sh
(lfs chroot) root:/sources/iproute2-5.8.0# cd ..
(lfs chroot) root:/sources# rm -rf iproute2-5.8.0
```

### 8.62 安装 Kbd-2.3.0

Kbd 包含键表文件、控制台字体和键盘实用程序。

解压软件包

```sh
(lfs chroot) root:/sources# tar xf kbd-2.3.0.tar.xz 
(lfs chroot) root:/sources# cd kbd-2.3.0
```

退格键和删除键的行为在 Kbd 包中的键映射中不一致。以下补丁修复了 i386 键盘映射的这个问题：

```sh
(lfs chroot) root:/sources/kbd-2.3.0# patch -Np1 -i ../kbd-2.3.0-backspace-1.patch
patching file data/keymaps/i386/dvorak/dvorak-l.map
patching file data/keymaps/i386/dvorak/dvorak-r.map
patching file data/keymaps/i386/fgGIod/tr_f-latin5.map
patching file data/keymaps/i386/qwerty/lt.l4.map
patching file data/keymaps/i386/qwerty/lt.map
patching file data/keymaps/i386/qwerty/no-latin1.map
patching file data/keymaps/i386/qwerty/ru1.map
patching file data/keymaps/i386/qwerty/ru2.map
patching file data/keymaps/i386/qwerty/ru-cp1251.map
patching file data/keymaps/i386/qwerty/ru-ms.map
patching file data/keymaps/i386/qwerty/ru_win.map
patching file data/keymaps/i386/qwerty/se-ir209.map
patching file data/keymaps/i386/qwerty/se-lat6.map
patching file data/keymaps/i386/qwerty/tr_q-latin5.map
patching file data/keymaps/i386/qwerty/ua.map
patching file data/keymaps/i386/qwerty/ua-utf.map
patching file data/keymaps/i386/qwerty/ua-utf-ws.map
patching file data/keymaps/i386/qwerty/ua-ws.map
```

打补丁后，退格键生成代码为 127 的字符，删除键生成众所周知的转义序列。

删除多余的 resizecons 程序（它需要已失效的 svgalib 来提供视频模式文件 - 对于正常使用，请适当设置控制台的 setfont 大小）及其手册页。

```sh
(lfs chroot) root:/sources/kbd-2.3.0# sed -i '/RESIZECONS_PROGS=/s/yes/no/' configure
(lfs chroot) root:/sources/kbd-2.3.0# sed -i 's/resizecons.8 //' docs/man/man8/Makefile.in
```

配置编译并检查：

```sh
time { ./configure --prefix=/usr --disable-vlock && make && make check; }

# 检查完成后显示内容如下：
## ------------- ##
## Test results. ##
## ------------- ##

36 tests were successful.
4 tests were skipped.
make[3]: Leaving directory '/sources/kbd-2.3.0/tests'
make[2]: Leaving directory '/sources/kbd-2.3.0/tests'
make[1]: Leaving directory '/sources/kbd-2.3.0/tests'
make[1]: Entering directory '/sources/kbd-2.3.0'
make[1]: Leaving directory '/sources/kbd-2.3.0'

real    0m43.283s
user    0m31.714s
sys     0m9.863s
```

安装软件包

```sh
time { make install; }

# 安装完成后显示内容如下：
make[3]: Nothing to be done for 'install-data-am'.
make[3]: Leaving directory '/sources/kbd-2.3.0/tests/libtswrap'
make[2]: Leaving directory '/sources/kbd-2.3.0/tests/libtswrap'
make[2]: Entering directory '/sources/kbd-2.3.0/tests'
make[3]: Entering directory '/sources/kbd-2.3.0/tests'
make[3]: Nothing to be done for 'install-exec-am'.
make[3]: Nothing to be done for 'install-data-am'.
make[3]: Leaving directory '/sources/kbd-2.3.0/tests'
make[2]: Leaving directory '/sources/kbd-2.3.0/tests'
make[1]: Leaving directory '/sources/kbd-2.3.0/tests'
make[1]: Entering directory '/sources/kbd-2.3.0'
make[2]: Entering directory '/sources/kbd-2.3.0'
make[2]: Nothing to be done for 'install-exec-am'.
make[2]: Nothing to be done for 'install-data-am'.
make[2]: Leaving directory '/sources/kbd-2.3.0'
make[1]: Leaving directory '/sources/kbd-2.3.0'

real    0m3.029s
user    0m1.790s
sys     0m0.833s
```

删除无意安装的内部库：

```sh
(lfs chroot) root:/sources/kbd-2.3.0# rm -v /usr/lib/libtswrap.{a,la,so*}
removed '/usr/lib/libtswrap.a'
removed '/usr/lib/libtswrap.la'
removed '/usr/lib/libtswrap.so'
removed '/usr/lib/libtswrap.so.1'
removed '/usr/lib/libtswrap.so.1.0.0'
```

注意：对于某些语言（例如，白俄罗斯语），Kbd 包不提供有用的键映射，其中库存“by”键映射假定 ISO-8859-5 编码，并且通常使用 CP1251 键映射。这些语言的用户必须单独下载工作键盘映射。

如果需要，请安装文档：

```sh
(lfs chroot) root:/sources/kbd-2.3.0# rm -v /usr/lib/libtswrap.{a,la,so*}
removed '/usr/lib/libtswrap.a'
removed '/usr/lib/libtswrap.la'
removed '/usr/lib/libtswrap.so'
removed '/usr/lib/libtswrap.so.1'
removed '/usr/lib/libtswrap.so.1.0.0'
(lfs chroot) root:/sources/kbd-2.3.0# mkdir -v /usr/share/doc/kbd-2.3.0
```

清理软件包

```sh
(lfs chroot) root:/sources/kbd-2.3.0# cd ..
(lfs chroot) root:/sources# rm -rf kbd-2.3.0
```

### 8.63 安装 Libpipeline-1.5.3

Libpipeline 包含一个以灵活方便的方式操作子进程管道的库。

解压软件包

```sh
(lfs chroot) root:/sources# tar xf libpipeline-1.5.3.tar.gz 
(lfs chroot) root:/sources# cd libpipeline-1.5.3
```

配置编译并检查：

```sh
time { ./configure --prefix=/usr && make && make check; }

# 检查完成后显示内容如下：
============================================================================
Testsuite summary for libpipeline 1.5.3
============================================================================
# TOTAL: 7
# PASS:  7
# SKIP:  0
# XFAIL: 0
# FAIL:  0
# XPASS: 0
# ERROR: 0
============================================================================
make[3]: Leaving directory '/sources/libpipeline-1.5.3/tests'
make[2]: Leaving directory '/sources/libpipeline-1.5.3/tests'
make[1]: Leaving directory '/sources/libpipeline-1.5.3/tests'
make[1]: Entering directory '/sources/libpipeline-1.5.3'
make[1]: Leaving directory '/sources/libpipeline-1.5.3'

real    0m45.876s
user    0m28.035s
sys     0m11.842s
```

安装软件包

```sh
time { make install; }

# 安装完成后显示内容如下：
make[1]: Leaving directory '/sources/libpipeline-1.5.3/tests'
make[1]: Entering directory '/sources/libpipeline-1.5.3'
make[2]: Entering directory '/sources/libpipeline-1.5.3'
make[2]: Nothing to be done for 'install-exec-am'.
make[2]: Nothing to be done for 'install-data-am'.
make[2]: Leaving directory '/sources/libpipeline-1.5.3'
make[1]: Leaving directory '/sources/libpipeline-1.5.3'

real    0m0.660s
user    0m0.375s
sys     0m0.185s
```

清理软件包

```sh
(lfs chroot) root:/sources/libpipeline-1.5.3# cd ..
(lfs chroot) root:/sources# rm -rf libpipeline-1.5.3
```

### 8.64 安装 Make-4.3

Make 包含一个程序，用于控制从源文件生成包的可执行文件和其他非源文件。

解压软件包

```sh
(lfs chroot) root:/sources# tar xf make-4.3.tar.gz 
(lfs chroot) root:/sources# cd make-4.3
```

配置编译并检查：

```sh
time { ./configure --prefix=/usr && make && make check; }

# 检查完成后显示内容如下：
690 Tests in 125 Categories Complete ... No Failures :-)

======================================================================
 Regression PASSED: GNU Make 4.3 (x86_64-pc-linux-gnu) built with gcc 
======================================================================

make[2]: Leaving directory '/sources/make-4.3'
make[1]: Leaving directory '/sources/make-4.3'

real    1m13.874s
user    0m21.730s
sys     0m8.474s
```

安装软件包

```sh
time { make install; }

# 安装完成后显示内容如下：
make[2]: Leaving directory '/sources/make-4.3'
make[1]: Leaving directory '/sources/make-4.3'

real    0m0.759s
user    0m0.393s
sys     0m0.240s
```

清理软件包

```sh
(lfs chroot) root:/sources/make-4.3# cd ..
(lfs chroot) root:/sources# rm -rf make-4.3
```

### 8.65 安装 Patch-2.7.6

Patch 包含一个程序，用于通过应用通常由 diff 程序创建的“补丁”文件来修改或创建文件。

解压软件包

```sh
(lfs chroot) root:/sources# tar xf patch-2.7.6.tar.xz 
(lfs chroot) root:/sources# cd patch-2.7.6
```

配置编译并检查：

```sh
time { ./configure --prefix=/usr && make && make check; }

# 检查完成后显示内容如下：
============================================================================
Testsuite summary for GNU patch 2.7.6
============================================================================
# TOTAL: 44
# PASS:  41
# SKIP:  1
# XFAIL: 2
# FAIL:  0
# XPASS: 0
# ERROR: 0
============================================================================
make[4]: Leaving directory '/sources/patch-2.7.6/tests'
make[3]: Leaving directory '/sources/patch-2.7.6/tests'
make[2]: Leaving directory '/sources/patch-2.7.6/tests'
make[2]: Entering directory '/sources/patch-2.7.6'
make[2]: Leaving directory '/sources/patch-2.7.6'
make[1]: Leaving directory '/sources/patch-2.7.6'

real    1m0.789s
user    0m36.871s
sys     0m15.860s
```

安装软件包

```sh
time { make install; }

# 安装完成后显示内容如下：
make[3]: Leaving directory '/sources/patch-2.7.6'
make[2]: Leaving directory '/sources/patch-2.7.6'
make[1]: Leaving directory '/sources/patch-2.7.6'

real    0m0.671s
user    0m0.400s
sys     0m0.179s
```

清理软件包

```sh
(lfs chroot) root:/sources/patch-2.7.6# cd ..
(lfs chroot) root:/sources# rm -rf patch-2.7.6
```

### 8.66 安装 Man-DB-2.9.3

Man-DB 包含用于查找和查看手册页的程序。

解压软件包

```sh
(lfs chroot) root:/sources# tar xf man-db-2.9.3.tar.xz 
(lfs chroot) root:/sources# cd man-db-2.9.3
```

配置编译并检查：

```sh
time { ./configure --prefix=/usr                 \
            --docdir=/usr/share/doc/man-db-2.9.3 \
            --sysconfdir=/etc                    \
            --disable-setuid                     \
            --enable-cache-owner=bin             \
            --with-browser=/usr/bin/lynx         \
            --with-vgrind=/usr/bin/vgrind        \
            --with-grap=/usr/bin/grap            \
            --with-systemdtmpfilesdir=           \
            --with-systemdsystemunitdir= && make && make check; }

# 检查完成后显示内容如下：
============================================================================
Testsuite summary for man-db 2.9.3
============================================================================
# TOTAL: 12
# PASS:  12
# SKIP:  0
# XFAIL: 0
# FAIL:  0
# XPASS: 0
# ERROR: 0
============================================================================
make[4]: Leaving directory '/sources/man-db-2.9.3/man'
make[3]: Leaving directory '/sources/man-db-2.9.3/man'
make[2]: Leaving directory '/sources/man-db-2.9.3/man'
make[1]: Leaving directory '/sources/man-db-2.9.3/man'
Making check in manual
make[1]: Entering directory '/sources/man-db-2.9.3/manual'
make[1]: Nothing to be done for 'check'.
make[1]: Leaving directory '/sources/man-db-2.9.3/manual'
Making check in po
make[1]: Entering directory '/sources/man-db-2.9.3/po'
make[1]: Nothing to be done for 'check'.
make[1]: Leaving directory '/sources/man-db-2.9.3/po'
Making check in tools
make[1]: Entering directory '/sources/man-db-2.9.3/tools'
make[1]: Nothing to be done for 'check'.
make[1]: Leaving directory '/sources/man-db-2.9.3/tools'
make[1]: Entering directory '/sources/man-db-2.9.3'
make[1]: Leaving directory '/sources/man-db-2.9.3'

real    2m8.851s
user    1m24.686s
sys     0m27.355s
```

安装软件包

```sh
time { make install; }

# 安装完成后显示内容如下：
Making install in tools
make[1]: Entering directory '/sources/man-db-2.9.3/tools'
make[2]: Entering directory '/sources/man-db-2.9.3/tools'
make[2]: Nothing to be done for 'install-exec-am'.
make[2]: Nothing to be done for 'install-data-am'.
make[2]: Leaving directory '/sources/man-db-2.9.3/tools'
make[1]: Leaving directory '/sources/man-db-2.9.3/tools'
make[1]: Entering directory '/sources/man-db-2.9.3'
make[2]: Entering directory '/sources/man-db-2.9.3'
make[2]: Nothing to be done for 'install-exec-am'.
make[2]: Nothing to be done for 'install-data-am'.
make[2]: Leaving directory '/sources/man-db-2.9.3'
make[1]: Leaving directory '/sources/man-db-2.9.3'

real    0m3.276s
user    0m2.026s
sys     0m0.843s
```

支持 `Simplified Chinese (zh_CN)` 语言，编码格式为 `GBK`

清理软件包

```sh
(lfs chroot) root:/sources/man-db-2.9.3# cd ..
(lfs chroot) root:/sources# rm -rf man-db-2.9.3
```

### 8.67 安装 Tar-1.32

Tar 提供了创建 tar 档案以及执行各种其他类型的档案操作的能力。Tar 可用于先前创建的档案以提取文件、存储其他文件，或者更新或列出已存储的文件。

解压软件包

```sh
(lfs chroot) root:/sources# tar xf tar-1.32.tar.xz 
(lfs chroot) root:/sources# cd tar-1.32
```

配置编译并检查：

```sh
time { FORCE_UNSAFE_CONFIGURE=1  \
./configure --prefix=/usr        \
            --bindir=/bin && make && make check; }

# 检查完成后显示内容如下：
## ------------- ##
## Test results. ##
## ------------- ##

ERROR: 215 tests were run,
1 failed unexpectedly.
19 tests were skipped.
## -------------------------- ##
## testsuite.log was created. ##
## -------------------------- ##

Please send `tests/testsuite.log' and all information you think might help:

   To: <bug-tar@gnu.org>
   Subject: [GNU tar 1.32] testsuite: 223 failed

You may investigate any problem if you feel able to do so, in which
case the test suite provides a good starting point.  Its output may
be found below `tests/testsuite.dir'.

make[2]: *** [Makefile:1838: check-local] Error 1
make[2]: Leaving directory '/sources/tar-1.32/tests'
make[1]: *** [Makefile:1690: check-am] Error 2
make[1]: Leaving directory '/sources/tar-1.32/tests'
make: *** [Makefile:1410: check-recursive] Error 1

real    7m52.237s
user    1m35.192s
sys     3m29.130s
```

有一项测试 `功能：二进制存储/恢复` (`capabilities: binary store/restore`)，已知会失败，可忽略。

失败的测试项：

```sh
Extended attributes

211: xattrs: basic functionality                     ok
212: xattrs: change directory with -C option         ok
213: xattrs: trusted.* attributes                    ok
214: xattrs: s/open/openat/ regression               ok
215: xattrs: keywords with '=' and '%'               ok
216: xattrs: exclude xattrs on create                ok
217: xattrs: xattrs and --skip-old-files             ok
218: acls: basic functionality                       ok
219: acls: work with -C                              ok
220: acls: default ACLs                              ok
221: selinux: basic store/restore                    skipped (selnx01.at:28)
222: acls/selinux: special files & fifos             skipped (selacl01.at:28)
223: capabilities: binary store/restore              FAILED (capabs_raw01.at:28)
```

安装软件包

```sh
time { make install && make -C doc install-html docdir=/usr/share/doc/tar-1.32; }

# 安装完成后显示内容如下：
Making install in tests
make[1]: Entering directory '/sources/tar-1.32/tests'
make[2]: Entering directory '/sources/tar-1.32/tests'
make[2]: Nothing to be done for 'install-exec-am'.
make[2]: Nothing to be done for 'install-data-am'.
make[2]: Leaving directory '/sources/tar-1.32/tests'
make[1]: Leaving directory '/sources/tar-1.32/tests'
make[1]: Entering directory '/sources/tar-1.32'
make[2]: Entering directory '/sources/tar-1.32'
make[2]: Nothing to be done for 'install-exec-am'.
make[2]: Nothing to be done for 'install-data-am'.
make[2]: Leaving directory '/sources/tar-1.32'
make[1]: Leaving directory '/sources/tar-1.32'
make: Entering directory '/sources/tar-1.32/doc'
  MAKEINFO tar.html
 /bin/mkdir -p '/usr/share/doc/tar-1.32'
 /bin/mkdir -p '/usr/share/doc/tar-1.32/tar.html'
 /usr/bin/install -c -m 644 'tar.html'/* '/usr/share/doc/tar-1.32/tar.html'
make: Leaving directory '/sources/tar-1.32/doc'

real    0m6.724s
user    0m5.790s
sys     0m0.619s
```

清理软件包

```sh
(lfs chroot) root:/sources/tar-1.32# cd ..
(lfs chroot) root:/sources# rm -rf tar-1.32
```

### 8.68 安装 Texinfo-6.7

Texinfo 包含用于读取、写入和转换信息页面的程序。

解压软件包

```sh
(lfs chroot) root:/sources# tar xf texinfo-6.7.tar.xz 
(lfs chroot) root:/sources# cd texinfo-6.7
```

配置编译并检查：

```sh
time { ./configure --prefix=/usr --disable-static && make && make check; }

# 检查完成后显示内容如下：
============================================================================
Testsuite summary for GNU Texinfo 6.7
============================================================================
# TOTAL: 1
# PASS:  1
# SKIP:  0
# XFAIL: 0
# FAIL:  0
# XPASS: 0
# ERROR: 0
============================================================================
make[3]: Leaving directory '/sources/texinfo-6.7/util'
make[2]: Leaving directory '/sources/texinfo-6.7/util'
make[1]: Leaving directory '/sources/texinfo-6.7/util'
Making check in doc
make[1]: Entering directory '/sources/texinfo-6.7/doc'
Making check in tp_api
make[2]: Entering directory '/sources/texinfo-6.7/doc/tp_api'
make  check-am
make[3]: Entering directory '/sources/texinfo-6.7/doc/tp_api'
make[3]: Nothing to be done for 'check-am'.
make[3]: Leaving directory '/sources/texinfo-6.7/doc/tp_api'
make[2]: Leaving directory '/sources/texinfo-6.7/doc/tp_api'
make[2]: Entering directory '/sources/texinfo-6.7/doc'
make[2]: Nothing to be done for 'check-am'.
make[2]: Leaving directory '/sources/texinfo-6.7/doc'
make[1]: Leaving directory '/sources/texinfo-6.7/doc'
Making check in man
make[1]: Entering directory '/sources/texinfo-6.7/man'
make[1]: Nothing to be done for 'check'.
make[1]: Leaving directory '/sources/texinfo-6.7/man'
make[1]: Entering directory '/sources/texinfo-6.7'
make[1]: Leaving directory '/sources/texinfo-6.7'

real    2m54.785s
user    2m17.273s
sys     0m30.475s
```

安装软件包，同时安装属于 TeX 的组件

```sh
time { make install && make TEXMF=/usr/share/texmf install-tex; }

# 安装完成后显示内容如下：
make[1]: Leaving directory '/sources/texinfo-6.7/doc'

real    0m2.606s
user    0m1.524s
sys     0m0.756s
```

信息文档系统使用纯文本文件来保存其菜单条目列表。该文件位于 `/usr/share/info/dir`。不幸的是，由于各种软件包的 Makefile 中偶尔出现问题，它有时会与系统上安装的信息页面不同步。如果需要重新创建 `/usr/share/info/dir` 文件，以下可选命令将完成任务：

```sh
# 执行以下脚本
pushd /usr/share/info
  rm -v dir
  for f in *
    do install-info $f dir 2>/dev/null
  done
popd
```

清理软件包

```sh
(lfs chroot) root:/sources/texinfo-6.7# cd ..
(lfs chroot) root:/sources# rm -rf texinfo-6.7
```

### 8.69 安装 Vim-8.2.1361

Vim 包含一个强大的文本编辑器。

#### 8.69.1 安装 Vim

解压软件包

```sh
(lfs chroot) root:/sources# tar xf vim-8.2.1361.tar.gz 
(lfs chroot) root:/sources# cd vim-8.2.1361
```

首先将 vimrc 配置文件的默认位置更改为 /etc：

```sh
(lfs chroot) root:/sources/vim-8.2.1361# echo '#define SYS_VIMRC_FILE "/etc/vimrc"' >> src/feature.h
```

配置并编译：

```sh
time { ./configure --prefix=/usr && make; }

# 编译完成后显示内容如下：
make[2]: Leaving directory '/sources/vim-8.2.1361/src/po'
make[2]: Entering directory '/sources/vim-8.2.1361/src/po'
make[2]: Nothing to be done for 'converted'.
make[2]: Leaving directory '/sources/vim-8.2.1361/src/po'
make[1]: Leaving directory '/sources/vim-8.2.1361/src'

real    2m41.459s
user    2m23.761s
sys     0m13.181s
```

要准备测试，请确保用户 tester 可以写入源代码树：

```sh
(lfs chroot) root:/sources/vim-8.2.1361# chown -Rv tester .
```

现在以用户 tester 身份运行测试：

```sh
(lfs chroot) root:/sources/vim-8.2.1361# su tester -c "LANG=en_US.UTF-8 make -j1 test" &> vim-test.log
```

测试套件向屏幕输出大量二进制数据。这可能会导致当前终端的设置出现问题。可以通过将输出重定向到日志文件来避免该问题，如上所示。成功的测试将在完成时在日志文件中显示 `ALL DONE` 字样。

输入 `cat vim-test.log ` 可以看到日志文件中的信息：

```sh
-------------------------------
Executed:  3139 Tests
 Skipped:    54 Tests
  Failed:     0 Tests

ALL DONE
make[2]: Leaving directory '/sources/vim-8.2.1361/src/testdir'
make[1]: Leaving directory '/sources/vim-8.2.1361/src'
make[1]: Entering directory '/sources/vim-8.2.1361'
```

安装软件包

```sh
time { make install; }

# 安装完成后显示内容如下：
make[1]: Leaving directory '/sources/vim-8.2.1361/src'

real    0m4.187s
user    0m1.746s
sys     0m1.528s
```

许多用户习惯于使用 vi 而不是 vim。要在用户习惯性地输入 vi 时允许执行 vim，请使用提供的语言为二进制文件和手册页创建一个符号链接：

```sh
(lfs chroot) root:/sources/vim-8.2.1361# ln -sv vim /usr/bin/vi
'/usr/bin/vi' -> 'vim'
# 继续执行以下脚本
for L in  /usr/share/man/{,*/}man1/vim.1; do
    ln -sv vim.1 $(dirname $L)/vi.1
done
# 输出以下内容：
'/usr/share/man/man1/vi.1' -> 'vim.1'
'/usr/share/man/da.ISO8859-1/man1/vi.1' -> 'vim.1'
'/usr/share/man/da.UTF-8/man1/vi.1' -> 'vim.1'
'/usr/share/man/da/man1/vi.1' -> 'vim.1'
'/usr/share/man/de.ISO8859-1/man1/vi.1' -> 'vim.1'
'/usr/share/man/de.UTF-8/man1/vi.1' -> 'vim.1'
'/usr/share/man/de/man1/vi.1' -> 'vim.1'
'/usr/share/man/fr.ISO8859-1/man1/vi.1' -> 'vim.1'
'/usr/share/man/fr.UTF-8/man1/vi.1' -> 'vim.1'
'/usr/share/man/fr/man1/vi.1' -> 'vim.1'
'/usr/share/man/it.ISO8859-1/man1/vi.1' -> 'vim.1'
'/usr/share/man/it.UTF-8/man1/vi.1' -> 'vim.1'
'/usr/share/man/it/man1/vi.1' -> 'vim.1'
'/usr/share/man/ja/man1/vi.1' -> 'vim.1'
'/usr/share/man/pl.ISO8859-2/man1/vi.1' -> 'vim.1'
'/usr/share/man/pl.UTF-8/man1/vi.1' -> 'vim.1'
'/usr/share/man/pl/man1/vi.1' -> 'vim.1'
'/usr/share/man/ru.KOI8-R/man1/vi.1' -> 'vim.1'
'/usr/share/man/ru.UTF-8/man1/vi.1' -> 'vim.1'
'/usr/share/man/tr.ISO8859-9/man1/vi.1' -> 'vim.1'
'/usr/share/man/tr.UTF-8/man1/vi.1' -> 'vim.1'
'/usr/share/man/tr/man1/vi.1' -> 'vim.1'
```

默认情况下，vim 的文档安装在 `/usr/share/vim` 中。以下符号链接允许通过 `/usr/share/doc/vim-8.2.1361` 访问文档，使其与其他包的文档位置保持一致：

```sh
(lfs chroot) root:/sources/vim-8.2.1361# ln -sv ../vim/vim82/doc /usr/share/doc/vim-8.2.1361
'/usr/share/doc/vim-8.2.1361' -> '../vim/vim82/doc'
```

清理软件包

```sh
(lfs chroot) root:/sources/vim-8.2.1361# cd ..
(lfs chroot) root:/sources# rm -rf vim-8.2.1361
```

#### 8.69.2 配置 Vim

默认情况下 vim 在不兼容 vi 的模式下运行。这对于过去使用其他编辑器的用户来说可能显得陌生。以下配置包含的 “nocompatible” 设定是为了强调编辑器使用了新的行为这一事实。它也提醒那些想要使用 “compatible” 模式的用户，必须在配置文件的一开始改变模式。这是因为它会修改其他设置，对这些设置的覆盖必须在设定模式后进行。执行以下命令创建默认 vim 配置文件：

```sh
# 执行以下指令
cat > /etc/vimrc << "EOF"
" Begin /etc/vimrc

" Ensure defaults are set before customizing settings, not after
source $VIMRUNTIME/defaults.vim
let skip_defaults_vim=1 

set nocompatible
set backspace=2
set mouse=
syntax on
if (&term == "xterm") || (&term == "putty")
  set background=dark
endif

" End /etc/vimrc
EOF
```

`set nocompatible` 设置使 vim 以比 vi 兼容方式更有用的方式（默认）运行。删除“no”以保留旧的 vi 行为。`set backspace=2` 设置允许在换行符、自动缩进和插入开始处退格。参数上的语法启用 vim 的语法突出显示。在 chroot 中或通过远程连接工作时，`set mouse=` 设置允许使用鼠标正确粘贴文本。最后，带有 `set background=dark` 设置的 if 语句纠正了 vim 对某些终端模拟器背景颜色的猜测。这为突出显示这些程序的黑色背景提供了更好的配色方案。

可以通过运行以下命令获取其他可用选项的文档：

```sh
vim -c ':options'
```

输入 `:q!` 可退出 Vim 界面

注意：默认情况下，vim 只安装英语的拼写文件。要为您的首选语言安装拼写文件，请从 ftp://ftp.vim.org/pub/vim/runtime/spell/ 下载 *.spl 和可选的 *.sug 文件，并保存它们到 /usr/share/vim/vim82/spell/。

要使用这些拼写文件，需要在 /etc/vimrc 中进行一些配置，例如：

```sh
# 在 /etc/vimrc 中配置 (我不运行此命令)
set spelllang=en,ru
set spell
```

有关更多信息，请参阅位于上述 URL 的相应 README 文件。

### 8.70 安装 Eudev-3.2.9

Eudev 包含用于动态创建设备节点的程序。

#### 8.70.1 安装 Eudev

解压软件包

```sh
(lfs chroot) root:/sources# tar xf eudev-3.2.9.tar.gz 
(lfs chroot) root:/sources# cd eudev-3.2.9
```

配置并编译：

```sh
time { ./configure --prefix=/usr    \
            --bindir=/sbin          \
            --sbindir=/sbin         \
            --libdir=/usr/lib       \
            --sysconfdir=/etc       \
            --libexecdir=/lib       \
            --with-rootprefix=      \
            --with-rootlibdir=/lib  \
            --enable-manpages       \
            --disable-static && make; }

# 编译完成后显示内容如下：
Making all in hwdb
make[2]: Entering directory '/sources/eudev-3.2.9/hwdb'
make[2]: Nothing to be done for 'all'.
make[2]: Leaving directory '/sources/eudev-3.2.9/hwdb'
make[2]: Entering directory '/sources/eudev-3.2.9'
make[2]: Leaving directory '/sources/eudev-3.2.9'
make[1]: Leaving directory '/sources/eudev-3.2.9'

real    0m42.277s
user    0m32.095s
sys     0m7.938s
```

现在创建一些测试所需的目录，但也将用作安装的一部分：

```sh
(lfs chroot) root:/sources/eudev-3.2.9# mkdir -pv /lib/udev/rules.d
mkdir: created directory '/lib/udev'
mkdir: created directory '/lib/udev/rules.d'
(lfs chroot) root:/sources/eudev-3.2.9# mkdir -pv /etc/udev/rules.d
mkdir: created directory '/etc/udev'
mkdir: created directory '/etc/udev/rules.d'
```

若要测试结果，请执行：

```sh
time { make check; }

# 检查完成后显示内容如下：
============================================================================
Testsuite summary for eudev 3.2.9
============================================================================
# TOTAL: 2
# PASS:  2
# SKIP:  0
# XFAIL: 0
# FAIL:  0
# XPASS: 0
# ERROR: 0
============================================================================
make[3]: Leaving directory '/sources/eudev-3.2.9/test'
make[2]: Leaving directory '/sources/eudev-3.2.9/test'
make[1]: Leaving directory '/sources/eudev-3.2.9/test'
Making check in man
make[1]: Entering directory '/sources/eudev-3.2.9/man'
make[1]: Nothing to be done for 'check'.
make[1]: Leaving directory '/sources/eudev-3.2.9/man'
Making check in hwdb
make[1]: Entering directory '/sources/eudev-3.2.9/hwdb'
make[1]: Nothing to be done for 'check'.
make[1]: Leaving directory '/sources/eudev-3.2.9/hwdb'
make[1]: Entering directory '/sources/eudev-3.2.9'
make[1]: Leaving directory '/sources/eudev-3.2.9'

real    0m2.457s
user    0m0.955s
sys     0m1.402s
```

安装软件包：

```sh
time { make install; }

# 安装完成后显示内容如下：
make[2]: Leaving directory '/sources/eudev-3.2.9/hwdb'
make[1]: Leaving directory '/sources/eudev-3.2.9/hwdb'
make[1]: Entering directory '/sources/eudev-3.2.9'
make[2]: Entering directory '/sources/eudev-3.2.9'
make[2]: Nothing to be done for 'install-exec-am'.
make[2]: Nothing to be done for 'install-data-am'.
make[2]: Leaving directory '/sources/eudev-3.2.9'
make[1]: Leaving directory '/sources/eudev-3.2.9'

real    0m2.028s
user    0m1.378s
sys     0m0.401s
```

安装一些在 LFS 环境中有用的自定义规则和支持文件：

```sh
(lfs chroot) root:/sources/eudev-3.2.9# tar -xvf ../udev-lfs-20171102.tar.xz
udev-lfs-20171102/
udev-lfs-20171102/init-net-rules.sh
udev-lfs-20171102/83-cdrom-symlinks.rules
udev-lfs-20171102/write_cd_rules
udev-lfs-20171102/ChangeLog
udev-lfs-20171102/81-cdrom.rules
udev-lfs-20171102/55-lfs.txt
udev-lfs-20171102/write_net_rules
udev-lfs-20171102/README
udev-lfs-20171102/contrib/
udev-lfs-20171102/contrib/debian/
udev-lfs-20171102/contrib/debian/83-cdrom-symlinks.rules
udev-lfs-20171102/contrib/debian/81-cdrom.rules
udev-lfs-20171102/contrib/debian/write_cd_aliases
udev-lfs-20171102/rule_generator.functions
udev-lfs-20171102/55-lfs.rules
udev-lfs-20171102/Makefile.lfs
(lfs chroot) root:/sources/eudev-3.2.9# make -f udev-lfs-20171102/Makefile.lfs install
mkdir: created directory '/usr/share/doc/udev-20171102'
mkdir: created directory '/usr/share/doc/udev-20171102/lfs'
'udev-lfs-20171102/55-lfs.rules' -> '/etc/udev/rules.d/55-lfs.rules'
'udev-lfs-20171102/81-cdrom.rules' -> '/etc/udev/rules.d/81-cdrom.rules'
'udev-lfs-20171102/83-cdrom-symlinks.rules' -> '/etc/udev/rules.d/83-cdrom-symlinks.rules'
'udev-lfs-20171102/write_cd_rules' -> '/lib/udev/write_cd_rules'
'udev-lfs-20171102/write_net_rules' -> '/lib/udev/write_net_rules'
'udev-lfs-20171102/init-net-rules.sh' -> '/lib/udev/init-net-rules.sh'
'udev-lfs-20171102/rule_generator.functions' -> '/lib/udev/rule_generator.functions'
'udev-lfs-20171102/README' -> '/usr/share/doc/udev-20171102/lfs/README'
'udev-lfs-20171102/55-lfs.txt' -> '/usr/share/doc/udev-20171102/lfs/55-lfs.txt'
```

清理安装包

```sh
(lfs chroot) root:/sources/eudev-3.2.9# cd ..
(lfs chroot) root:/sources# rm -rf eudev-3.2.9
```

#### 8.70.2 配置 Eudev

有关硬件设备的信息保存在 `/etc/udev/hwdb.d` 和 `/lib/udev/hwdb.d` 目录中。 Eudev 需要将该信息编译到二进制数据库 `/etc/udev/hwdb.bin` 中。创建初始数据库：

```sh
(lfs chroot) root:/sources# udevadm hwdb --update
```

每次更新硬件信息时都需要运行此命令。

### 8.71 安装 Procps-ng-3.3.16

Procps-ng 包含用于监控进程的程序。

解压软件包

```sh
(lfs chroot) root:/sources# tar xf procps-ng-3.3.16.tar.xz 
(lfs chroot) root:/sources# cd procps-ng-3.3.16
```

配置编译并检查：

```sh
time { ./configure --prefix=/usr                     \
            --exec-prefix=                           \
            --libdir=/usr/lib                        \
            --docdir=/usr/share/doc/procps-ng-3.3.16 \
            --disable-static                         \
            --disable-kill && make && make check; }

# 检查完成后显示内容如下：
============================================================================
Testsuite summary for procps-ng 3.3.16
============================================================================
# TOTAL: 1
# PASS:  1
# SKIP:  0
# XFAIL: 0
# FAIL:  0
# XPASS: 0
# ERROR: 0
============================================================================
make[4]: Leaving directory '/sources/procps-ng-3.3.16'
make[3]: Leaving directory '/sources/procps-ng-3.3.16'
make[2]: Leaving directory '/sources/procps-ng-3.3.16'
make[1]: Leaving directory '/sources/procps-ng-3.3.16'

real    0m46.880s
user    0m24.363s
sys     0m7.086s
```

安装软件包

```sh
time { make install; }

# 安装完成后显示内容如下：
make[3]: Leaving directory '/sources/procps-ng-3.3.16'
make[2]: Leaving directory '/sources/procps-ng-3.3.16'
make[1]: Leaving directory '/sources/procps-ng-3.3.16'

real    0m1.642s
user    0m1.001s
sys     0m0.525s
```

最后，将必要的库移动到一个可以在 /usr 未安装的情况下找到的位置。

```sh
(lfs chroot) root:/sources/procps-ng-3.3.16# mv -v /usr/lib/libprocps.so.* /lib
renamed '/usr/lib/libprocps.so.8' -> '/lib/libprocps.so.8'
renamed '/usr/lib/libprocps.so.8.0.2' -> '/lib/libprocps.so.8.0.2'
(lfs chroot) root:/sources/procps-ng-3.3.16# ln -sfv ../../lib/$(readlink /usr/lib/libprocps.so) /usr/lib/libprocps.so
'/usr/lib/libprocps.so' -> '../../lib/libprocps.so.8.0.2'
```

清理软件包

```sh
(lfs chroot) root:/sources/procps-ng-3.3.16# cd ..
(lfs chroot) root:/sources# rm -rf procps-ng-3.3.16
```

### 8.72 安装 Util-linux-2.36

Util-linux 包含各种实用程序。其中包括用于处理文件系统、控制台、分区和消息的实用程序。

解压软件包

```sh
(lfs chroot) root:/sources# tar xf util-linux-2.36.tar.xz 
(lfs chroot) root:/sources# cd util-linux-2.36
```

FHS 建议使用 `/var/lib/hwclock` 目录而不是通常的 `/etc` 目录作为 `adjtime` 文件的位置。使用以下命令创建此目录：

```sh
(lfs chroot) root:/sources/util-linux-2.36# mkdir -pv /var/lib/hwclock
```

配置并编译，`--disable` 和 `--without` 选项可防止有关构建需要不在 LFS 中的包或与其他包安装的程序不一致的组件的警告：

```sh
time { ./configure ADJTIME_PATH=/var/lib/hwclock/adjtime   \
            --docdir=/usr/share/doc/util-linux-2.36 \
            --disable-chfn-chsh  \
            --disable-login      \
            --disable-nologin    \
            --disable-su         \
            --disable-setpriv    \
            --disable-runuser    \
            --disable-pylibmount \
            --disable-static     \
            --without-python     \
            --without-systemd    \
            --without-systemdsystemunitdir && make; }

# 编译完成后显示内容如下：
make[2]: Leaving directory '/sources/util-linux-2.36'
make[1]: Leaving directory '/sources/util-linux-2.36'

real    3m20.357s
user    2m39.671s
sys     0m34.409s
```

警告：以 root 用户身份运行测试套件可能对您的系统有害。要运行它，内核的 `CONFIG_SCSI_DEBUG` 选项必须在当前运行的系统中可用，并且必须构建为模块。将其构建到内核中将阻止启动。为了完全覆盖，必须安装其他 BLFS 包。如果需要，可以在重新启动到完整的 LFS 系统并运行后运行此测试：`bash tests/run.sh --srcdir=$PWD --builddir=$PWD` (我不运行此命令)

如果需要，以非 root 用户身份运行测试套件：

```sh
(lfs chroot) root:/sources/util-linux-2.36# chown -Rv tester .
(lfs chroot) root:/sources/util-linux-2.36# su tester -c "make -k check"

# 测试套件的输出信息：
---------------------------------------------------------------------
  All 207 tests PASSED
---------------------------------------------------------------------
make[3]: Leaving directory '/sources/util-linux-2.36'
make[2]: Leaving directory '/sources/util-linux-2.36'
make[1]: Leaving directory '/sources/util-linux-2.36'
```

安装软件包

```sh
time { make install; }

# 安装完成后显示内容如下：
make  install-data-hook
make[4]: Entering directory '/sources/util-linux-2.36'
make[4]: Nothing to be done for 'install-data-hook'.
make[4]: Leaving directory '/sources/util-linux-2.36'
make[3]: Leaving directory '/sources/util-linux-2.36'
make[2]: Leaving directory '/sources/util-linux-2.36'
make[1]: Leaving directory '/sources/util-linux-2.36'

real    0m7.987s
user    0m5.014s
sys     0m2.237s
```

清理软件包

```sh
(lfs chroot) root:/sources/util-linux-2.36# cd ..
(lfs chroot) root:/sources# rm -rf util-linux-2.36
```

### 8.73 安装 E2fsprogs-1.45.6

E2fsprogs 包含用于处理 ext2 文件系统的实用程序。它还支持 ext3 和 ext4 日志文件系统。

解压软件包

```sh
(lfs chroot) root:/sources# tar xf e2fsprogs-1.45.6.tar.gz 
(lfs chroot) root:/sources# cd e2fsprogs-1.45.6
```

E2fsprogs 文档建议将包构建在源代码树的子目录中：

```sh
(lfs chroot) root:/sources/e2fsprogs-1.45.6# mkdir -v build
mkdir: created directory 'build'
(lfs chroot) root:/sources/e2fsprogs-1.45.6# cd build
```

配置编译并检查：

```sh
time { ../configure --prefix=/usr    \
             --bindir=/bin           \
             --with-root-prefix=""   \
             --enable-elf-shlibs     \
             --disable-libblkid      \
             --disable-libuuid       \
             --disable-uuidd         \
             --disable-fsck && make && make check; }

# 检查完成后显示内容如下：
357 tests succeeded     0 tests failed
Creating test_script...
make[1]: Leaving directory '/sources/e2fsprogs-1.45.6/build/tests'

real    4m34.993s
user    2m43.897s
sys     0m39.063s
```

在机械硬盘上，测试需要 4 个多一点的 SBU。它们在 SSD 上可以更短（低至约 1.5 个 SBU）。

安装软件包

```sh
time { make install; }

# 安装完成后显示内容如下：
make[2]: Leaving directory '/sources/e2fsprogs-1.45.6/build/intl'
make[1]: Leaving directory '/sources/e2fsprogs-1.45.6/build'

real    0m3.192s
user    0m1.583s
sys     0m1.009s
```

使已安装的静态库可写，以便以后可以删除调试符号：

```sh
(lfs chroot) root:/sources/e2fsprogs-1.45.6/build# chmod -v u+w /usr/lib/{libcom_err,libe2p,libext2fs,libss}.a
mode of '/usr/lib/libcom_err.a' changed from 0444 (r--r--r--) to 0644 (rw-r--r--)
mode of '/usr/lib/libe2p.a' changed from 0444 (r--r--r--) to 0644 (rw-r--r--)
mode of '/usr/lib/libext2fs.a' changed from 0444 (r--r--r--) to 0644 (rw-r--r--)
mode of '/usr/lib/libss.a' changed from 0444 (r--r--r--) to 0644 (rw-r--r--)
```

这个包安装了一个 gzip 压缩的 .info 文件，但不更新系统范围的 dir 文件。解压缩此文件，然后使用以下命令更新系统目录文件：

```sh
(lfs chroot) root:/sources/e2fsprogs-1.45.6/build# gunzip -v /usr/share/info/libext2fs.info.gz
/usr/share/info/libext2fs.info.gz:       79.7% -- replaced with /usr/share/info/libext2fs.info
(lfs chroot) root:/sources/e2fsprogs-1.45.6/build# install-info --dir-file=/usr/share/info/dir /usr/share/info/libext2fs.info
```

如果需要，通过执行以下命令来创建和安装一些附加文档：

```sh
(lfs chroot) root:/sources/e2fsprogs-1.45.6/build# makeinfo -o doc/com_err.info ../lib/et/com_err.texinfo
(lfs chroot) root:/sources/e2fsprogs-1.45.6/build# install -v -m644 doc/com_err.info /usr/share/info
'doc/com_err.info' -> '/usr/share/info/com_err.info'
(lfs chroot) root:/sources/e2fsprogs-1.45.6/build# install-info --dir-file=/usr/share/info/dir /usr/share/info/com_err.info
```

清除软件包

```sh
(lfs chroot) root:/sources/e2fsprogs-1.45.6/build# cd ../..
(lfs chroot) root:/sources# rm -rf e2fsprogs-1.45.6
```

### 8.74 安装 Sysklogd-1.5.1

Sysklogd 包含记录系统消息的程序，例如发生异常情况时内核给出的消息。

#### 8.74.1 安装 Sysklogd

解压软件包

```sh
(lfs chroot) root:/sources# tar xf sysklogd-1.5.1.tar.gz 
(lfs chroot) root:/sources# cd sysklogd-1.5.1
```

首先修复在 klogd 中某些条件下导致分段错误的问题，并修复一个过时的程序结构：

```sh
(lfs chroot) root:/sources/sysklogd-1.5.1# sed -i '/Error loading kernel symbols/{n;n;d}' ksym_mod.c
(lfs chroot) root:/sources/sysklogd-1.5.1# sed -i 's/union wait/int/' syslogd.c
```

编译并安装软件包

```sh
time { make && make BINDIR=/sbin install; }

# 安装完成后输出以下信息:
/usr/bin/install -o root -g root -m 644 sysklogd.8 /usr/share/man/man8/sysklogd.8
/usr/bin/install -o root -g root -m 644 syslogd.8 /usr/share/man/man8/syslogd.8
/usr/bin/install -o root -g root -m 644 syslog.conf.5 /usr/share/man/man5/syslog.conf.5
/usr/bin/install -o root -g root -m 644 klogd.8 /usr/share/man/man8/klogd.8
/usr/bin/install -m 500 -s syslogd /sbin/syslogd
/usr/bin/install -m 500 -s klogd /sbin/klogd

real    0m1.933s
user    0m1.644s
sys     0m0.209s
```

#### 8.74.2 配置 Sysklogd

通过运行以下命令创建一个新的 `/etc/syslog.conf` 文件：

```sh
# 执行以下命令：
cat > /etc/syslog.conf << "EOF"
# Begin /etc/syslog.conf

auth,authpriv.* -/var/log/auth.log
*.*;auth,authpriv.none -/var/log/sys.log
daemon.* -/var/log/daemon.log
kern.* -/var/log/kern.log
mail.* -/var/log/mail.log
user.* -/var/log/user.log
*.emerg *

# End /etc/syslog.conf
EOF
```

清除软件包

```sh
(lfs chroot) root:/sources/sysklogd-1.5.1# cd ..
(lfs chroot) root:/sources# rm -rf sysklogd-1.5.1
```

### 8.75 安装 Sysvinit-2.97

Sysvinit 包含用于控制系统启动、运行和关闭的程序。

解压软件包

```sh
(lfs chroot) root:/sources# tar xf sysvinit-2.97.tar.xz 
(lfs chroot) root:/sources# cd sysvinit-2.97
```

首先应用一个补丁来删除其他包安装的几个程序，澄清一条消息，并修复一个编译器警告：

```sh
(lfs chroot) root:/sources/sysvinit-2.97# patch -Np1 -i ../sysvinit-2.97-consolidated-1.patch
patching file src/Makefile
Hunk #2 succeeded at 211 (offset 2 lines).
Hunk #3 succeeded at 236 (offset 2 lines).
```

编译并安装软件包

```sh
time { make && make install; }

# 安装完成后输出以下信息:
make[1]: Leaving directory '/sources/sysvinit-2.97/src'

real    0m3.634s
user    0m2.978s
sys     0m0.478s
```

清除软件包

```sh
(lfs chroot) root:/sources/sysvinit-2.97# cd ..
(lfs chroot) root:/sources# rm -rf sysvinit-2.97
```

### 8.76 关于调试符号

默认情况下，大多数程序和库在编译时都包含调试符号（使用 gcc 的 -g 选项）。这意味着在调试使用调试信息编译的程序或库时，调试器不仅可以提供内存地址，还可以提供例程和变量的名称。

但是，包含这些调试符号会显著扩大程序或库。以下是这些符号占用空间量的示例：

* 带有调试符号的 bash 二进制文件：1200 KB
* 没有调试符号的 bash 二进制文件：480 KB
* 带有调试符号的 Glibc 和 GCC 文件（/lib 和/usr/lib）：87 MB
* 不带调试符号的 Glibc 和 GCC 文件：16 MB

大小可能因使用的编译器和 C 库而异，但在比较有和没有调试符号的程序时，差异通常是 2 到 5 倍之间。

因为大多数用户永远不会在他们的系统软件上使用调试器，所以通过删除这些符号可以重新获得大量磁盘空间。下一节将展示如何从程序和库中去除所有调试符号。

### 8.77 移除调试符号

**本节是可选的**。如果目标用户不是程序员并且不打算对系统软件进行任何调试，则通过从二进制文件和库中删除调试符号，可以将系统大小减少约 2 GB。**除了无法再完全调试软件之外，这不会造成任何不便**。

大多数使用下面提到的命令的人不会遇到任何困难。但是，很容易打错字并使新系统无法使用，因此在运行 strip 命令之前，**最好在当前状态下备份 LFS 系统**。

首先将所选库的调试符号放在单独的文件中。如果稍后在 BLFS 中运行使用 valgrind 或 gdb 的回归测试，则需要此调试信息。

‼️ **强烈建议在进行下面的步骤前，先使用 *快照* 备份你的虚拟机！**

```sh
# 备份完成后，依次执行以下命令：

save_lib="ld-2.32.so libc-2.32.so libpthread-2.32.so libthread_db-1.0.so"

cd /lib

for LIB in $save_lib; do
    objcopy --only-keep-debug $LIB $LIB.dbg 
    strip --strip-unneeded $LIB
    objcopy --add-gnu-debuglink=$LIB.dbg $LIB 
done

save_usrlib="libquadmath.so.0.0.0 libstdc++.so.6.0.28
             libitm.so.1.0.0 libatomic.so.1.2.0" 

cd /usr/lib

for LIB in $save_usrlib; do
    objcopy --only-keep-debug $LIB $LIB.dbg
    strip --strip-unneeded $LIB
    objcopy --add-gnu-debuglink=$LIB.dbg $LIB
done

unset LIB save_lib save_usrlib
```

现在可以移除调试符号：

```sh
find /usr/lib -type f -name \*.a \
   -exec strip --strip-debug {} ';'

find /lib /usr/lib -type f -name \*.so* ! -name \*dbg \
   -exec strip --strip-unneeded {} ';'

find /{bin,sbin} /usr/{bin,sbin,libexec} -type f \
    -exec strip --strip-all {} ';'
```

大量文件将被报告为**无法识别其文件格式**。这些警告可以安全地**忽略**。这些警告表明这些文件是脚本而不是二进制文件。

```sh
# 以下是我运行上述命令时的输出信息： (以下不要重复运行)
(lfs chroot) root:/usr/lib# find /usr/lib -type f -name \*.a \
>    -exec strip --strip-debug {} ';'
strip: /usr/lib/libm.a: file format not recognized
(lfs chroot) root:/usr/lib# find /lib /usr/lib -type f -name \*.so* ! -name \*dbg \
>    -exec strip --strip-unneeded {} ';'
strip: /usr/lib/libcursesw.so: file format not recognized
strip: /usr/lib/libc.so: file format not recognized
strip: /usr/lib/libncurses.so: file format not recognized
strip: /usr/lib/libform.so: file format not recognized
strip: /usr/lib/libm.so: file format not recognized
strip: /usr/lib/libgcc_s.so: file format not recognized
strip: /usr/lib/libpanel.so: file format not recognized
strip: /usr/lib/libmenu.so: file format not recognized
(lfs chroot) root:/usr/lib# find /{bin,sbin} /usr/{bin,sbin,libexec} -type f \
>     -exec strip --strip-all {} ';'
strip: /bin/egrep: file format not recognized
strip: /bin/compile_et: file format not recognized
strip: /bin/mk_cmds: file format not recognized
strip: /bin/fgrep: file format not recognized
strip: /sbin/routel: file format not recognized
strip: /sbin/e2scrub: file format not recognized
strip: /sbin/e2scrub_all: file format not recognized
strip: /sbin/ifcfg: file format not recognized
strip: /sbin/rtpr: file format not recognized
strip: /sbin/grub-set-default: file format not recognized
strip: /sbin/routef: file format not recognized
strip: /sbin/grub-reboot: file format not recognized
strip: /sbin/grub-mkconfig: file format not recognized
strip: /usr/bin/glilypond: file format not recognized
strip: /usr/bin/znew: file format not recognized
strip: /usr/bin/decryptdir: file format not recognized
strip: /usr/bin/multixterm: file format not recognized
strip: /usr/bin/pod2text: file format not recognized
strip: /usr/bin/prove: file format not recognized
strip: /usr/bin/automake: file format not recognized
strip: /usr/bin/bashbug: file format not recognized
strip: /usr/bin/gdiffmk: file format not recognized
strip: /usr/bin/xtrace: file format not recognized
strip: /usr/bin/autoconf: file format not recognized
strip: /usr/bin/autoexpect: file format not recognized
strip: /usr/bin/xpstat: file format not recognized
strip: /usr/bin/gpinyin: file format not recognized
strip: /usr/bin/pl2pm: file format not recognized
strip: /usr/bin/texi2any: file format not recognized
strip: /usr/bin/intltool-prepare: file format not recognized
strip: /usr/bin/updatedb: file format not recognized
strip: /usr/bin/enc2xs: file format not recognized
strip: /usr/bin/xzless: file format not recognized
strip: /usr/bin/zmore: file format not recognized
strip: /usr/bin/piconv: file format not recognized
strip: /usr/bin/encguess: file format not recognized
strip: /usr/bin/zforce: file format not recognized
strip: /usr/bin/roff2dvi: file format not recognized
strip: /usr/bin/gunzip: file format not recognized
strip: /usr/bin/xsubpp: file format not recognized
strip: /usr/bin/zcmp: file format not recognized
strip: /usr/bin/pic2graph: file format not recognized
strip: /usr/bin/rftp: file format not recognized
strip: /usr/bin/aclocal: file format not recognized
strip: /usr/bin/zdiff: file format not recognized
strip: /usr/bin/timed-read: file format not recognized
strip: /usr/bin/unicode_start: file format not recognized
strip: /usr/bin/bzgrep: file format not recognized
strip: /usr/bin/pydoc3.8: file format not recognized
strip: /usr/bin/roff2x: file format not recognized
strip: /usr/bin/shasum: file format not recognized
strip: /usr/bin/sotruss: file format not recognized
strip: /usr/bin/xzmore: file format not recognized
strip: /usr/bin/kibitz: file format not recognized
strip: /usr/bin/yacc: file format not recognized
strip: /usr/bin/neqn: file format not recognized
strip: /usr/bin/pod2man: file format not recognized
strip: /usr/bin/timed-run: file format not recognized
strip: /usr/bin/xkibitz: file format not recognized
strip: /usr/bin/texindex: file format not recognized
strip: /usr/bin/autoupdate: file format not recognized
strip: /usr/bin/gperl: file format not recognized
strip: /usr/bin/automake-1.16: file format not recognized
strip: /usr/bin/cryptdir: file format not recognized
strip: /usr/bin/perlthanks: file format not recognized
strip: /usr/bin/ncursesw6-config: file format not recognized
strip: /usr/bin/intltoolize: file format not recognized
strip: /usr/bin/dislocate: file format not recognized
strip: /usr/bin/gettextize: file format not recognized
strip: /usr/bin/grog: file format not recognized
strip: /usr/bin/mkpasswd: file format not recognized
strip: /usr/bin/h2xs: file format not recognized
strip: /usr/bin/grap2graph: file format not recognized
strip: /usr/bin/pdfmom: file format not recognized
strip: /usr/bin/roff2pdf: file format not recognized
strip: /usr/bin/instmodsh: file format not recognized
strip: /usr/bin/python3.8-config: file format not recognized
strip: /usr/bin/ifnames: file format not recognized
strip: /usr/bin/roff2html: file format not recognized
strip: /usr/bin/pod2texi: file format not recognized
strip: /usr/bin/mmroff: file format not recognized
strip: /usr/bin/zfgrep: file format not recognized
strip: /usr/bin/autopasswd: file format not recognized
strip: /usr/bin/perlbug: file format not recognized
strip: /usr/bin/autom4te: file format not recognized
strip: /usr/bin/pdftexi2dvi: file format not recognized
strip: /usr/bin/texi2pdf: file format not recognized
strip: /usr/bin/groffer: file format not recognized
strip: /usr/bin/splain: file format not recognized
strip: /usr/bin/passmass: file format not recognized
strip: /usr/bin/meson: file format not recognized
strip: /usr/bin/intltool-merge: file format not recognized
strip: /usr/bin/autoreconf: file format not recognized
strip: /usr/bin/roff2ps: file format not recognized
strip: /usr/bin/pip3.8: file format not recognized
strip: /usr/bin/autoheader: file format not recognized
strip: /usr/bin/c_rehash: file format not recognized
strip: /usr/bin/libnetcfg: file format not recognized
strip: /usr/bin/gettext.sh: file format not recognized
strip: /usr/bin/rlogin-cwd: file format not recognized
strip: /usr/bin/h2ph: file format not recognized
strip: /usr/bin/ptar: file format not recognized
strip: /usr/bin/libtool: file format not recognized
strip: /usr/bin/zgrep: file format not recognized
strip: /usr/bin/libtoolize: file format not recognized
strip: /usr/bin/unbuffer: file format not recognized
strip: /usr/bin/zegrep: file format not recognized
strip: /usr/bin/roff2text: file format not recognized
strip: /usr/bin/gzexe: file format not recognized
strip: /usr/bin/texi2dvi: file format not recognized
strip: /usr/bin/unicode_stop: file format not recognized
strip: /usr/bin/corelist: file format not recognized
strip: /usr/bin/catchsegv: file format not recognized
strip: /usr/bin/eqn2graph: file format not recognized
strip: /usr/bin/autopoint: file format not recognized
strip: /usr/bin/gropdf: file format not recognized
strip: /usr/bin/perlivp: file format not recognized
strip: /usr/bin/xzdiff: file format not recognized
strip: /usr/bin/checkmk: file format not recognized
strip: /usr/bin/aclocal-1.16: file format not recognized
strip: /usr/bin/json_pp: file format not recognized
strip: /usr/bin/easy_install-3.8: file format not recognized
strip: /usr/bin/ptargrep: file format not recognized
strip: /usr/bin/pod2html: file format not recognized
strip: /usr/bin/cpan: file format not recognized
strip: /usr/bin/pod2usage: file format not recognized
strip: /usr/bin/zstdless: file format not recognized
strip: /usr/bin/perldoc: file format not recognized
strip: /usr/bin/mtrace: file format not recognized
strip: /usr/bin/ptardiff: file format not recognized
strip: /usr/bin/tkpasswd: file format not recognized
strip: /usr/bin/nroff: file format not recognized
strip: /usr/bin/idle3.8: file format not recognized
strip: /usr/bin/intltool-extract: file format not recognized
strip: /usr/bin/ftp-rfc: file format not recognized
strip: /usr/bin/zless: file format not recognized
strip: /usr/bin/ldd: file format not recognized
strip: /usr/bin/intltool-update: file format not recognized
strip: /usr/bin/grub-kbdcomp: file format not recognized
strip: /usr/bin/vimtutor: file format not recognized
strip: /usr/bin/weather: file format not recognized
strip: /usr/bin/zstdgrep: file format not recognized
strip: /usr/bin/uncompress: file format not recognized
strip: /usr/bin/zipdetails: file format not recognized
strip: /usr/bin/2to3-3.8: file format not recognized
strip: /usr/bin/lpunlock: file format not recognized
strip: /usr/bin/podchecker: file format not recognized
strip: /usr/bin/xzgrep: file format not recognized
strip: /usr/bin/bzmore: file format not recognized
strip: /usr/bin/pdfroff: file format not recognized
strip: /usr/bin/streamzip: file format not recognized
strip: /usr/bin/runtest: file format not recognized
strip: /usr/bin/bzdiff: file format not recognized
strip: /usr/bin/zcat: file format not recognized
strip: /usr/bin/tzselect: file format not recognized
strip: /usr/bin/autoscan: file format not recognized
strip: /usr/bin/chem: file format not recognized
strip: /usr/bin/afmtodit: file format not recognized
strip: /usr/bin/tknewsbiff: file format not recognized
strip: /usr/bin/sqlite3_analyzer: file format not recognized
strip: /usr/libexec/gcc/x86_64-pc-linux-gnu/10.2.0/install-tools/fixinc.sh: file format not recognized
strip: /usr/libexec/gcc/x86_64-pc-linux-gnu/10.2.0/install-tools/mkinstalldirs: file format not recognized
strip: /usr/libexec/gcc/x86_64-pc-linux-gnu/10.2.0/install-tools/mkheaders: file format not recognized
strip: /usr/libexec/gcc/x86_64-pc-linux-gnu/10.2.0/liblto_plugin.la: file format not recognized
strip: /usr/libexec/gcc/x86_64-lfs-linux-gnu/10.2.0/install-tools/fixinc.sh: file format not recognized
strip: /usr/libexec/gcc/x86_64-lfs-linux-gnu/10.2.0/install-tools/mkinstalldirs: file format not recognized
strip: /usr/libexec/gcc/x86_64-lfs-linux-gnu/10.2.0/install-tools/mkheaders: file format not recognized
(lfs chroot) root:/usr/lib# 
```

### 8.78 清理

现在，可以使用 `df -h /` 查看一下磁盘所占用的空间，以便于清理之后进行对比。

```sh
(lfs chroot) root:/usr/lib# df -h /       
Filesystem      Size  Used Avail Use% Mounted on
/dev/sdb1        20G  2.4G   17G  13% /
```

最后，清理运行测试留下的一些额外文件：

```sh
(lfs chroot) root:/usr/lib# rm -rf /tmp/*
```

现在注销并使用新的 chroot 命令重新进入 chroot 环境。从现在开始，在退出后需要重新进入 chroot 环境时，退出之后请使用下面新的 chroot 命令：

如果虚拟内核文件系统已被卸载（手动或通过重新引导），**请确保在重新进入 chroot 时已安装虚拟内核文件系统**。此过程在 第7.3.2节-安装和填充/dev 和 第7.3.3节-安装虚拟内核文件系统 中进行了解释。

```sh
# 退出 chroot 环境
(lfs chroot) root:/usr/lib# logout
root@ubuntu:~# 

# 在执行进入 chroot 环境的步骤前，为了确保已经挂载了虚拟内核文件系统，请执行以下命令
root@ubuntu:~# ls $LFS
bin   dev  home  lib64       media  opt   root  sbin     srv  tmp    usr
boot  etc  lib   lost+found  mnt    proc  run   sources  sys  tools  var
# 如果可以显示以上文件，说明虚拟内核文件系统已被正确挂载，否则，请参考 [9.14.3 恢复] 章节重新挂载！！

# 使用新的命令进入 chroot 环境
chroot "$LFS" /usr/bin/env -i          \
    HOME=/root TERM="$TERM"            \
    PS1='(lfs chroot) \u:\w\$ '        \
    PATH=/bin:/usr/bin:/sbin:/usr/sbin \
    /bin/bash --login
```

这里不再使用 +h 选项，因为所有以前的程序都已被替换：因此可以进行散列。

为了满足几个包中的回归测试，本章前面没有抑制几个静态库。这些库来自 binutils、bzip2、e2fsprogs、flex、libtool 和 zlib。如果需要，现在可以删除它们：

```sh
(lfs chroot) root:/# rm -f /usr/lib/lib{bfd,opcodes}.a
(lfs chroot) root:/# rm -f /usr/lib/libctf{,-nobfd}.a
(lfs chroot) root:/# rm -f /usr/lib/libbz2.a
(lfs chroot) root:/# rm -f /usr/lib/lib{com_err,e2p,ext2fs,ss}.a
(lfs chroot) root:/# rm -f /usr/lib/libltdl.a
(lfs chroot) root:/# rm -f /usr/lib/libfl.a
(lfs chroot) root:/# rm -f /usr/lib/libz.a
```

在 `/usr/lib` 和 `/usr/libexec` 目录中还安装了几个文件扩展名为 `.la` 的文件。这些是 "libtool 归档" 文件。如前所述，它们仅在与静态库链接时才有用。当使用动态共享库时，尤其是在使用非自动工具构建系统时，它们是不需要的，并且可能有害。要删除它们，请运行：

```sh
(lfs chroot) root:/# find /usr/lib /usr/libexec -name \*.la -delete
```

有关 libtool 存档文件的更多信息，请参阅 BLFS 部分 "关于 Libtool 存档 (.la) 文件" 。

第 6 章和第 7 章中仍然安装了一些构建时的编译器，现在不再需要它们。删除它：

```sh
(lfs chroot) root:/# find /usr -depth -name $(uname -m)-lfs-linux-gnu\* | xargs rm -rf
```

也可以删除 `/tools` 目录以进一步获得一些空间：

```sh
(lfs chroot) root:/# rm -rf /tools
```

最后，删除在上一章开头创建的临时 `tester` 用户帐户。

```sh
(lfs chroot) root:/# userdel -r tester
userdel: tester mail spool (/var/mail/tester) not found
```

系统清理终于结束了！！

现在，可以使用 `df -h /` 查看一下磁盘所占用的空间。

```sh
(lfs chroot) root:/# df -h /
Filesystem      Size  Used Avail Use% Mounted on
/dev/sdb1        20G  1.6G   17G   9% /
```

**强烈建议再次使用 *快照功能* 进行备份**。

## 第九章 系统配置

### 9.1 概述

引导 Linux 系统涉及多项任务。该进程必须挂载虚拟和真实文件系统，初始化设备，激活交换，检查文件系统的完整性，设置系统时钟，启动网络，启动系统所需的任何守护程序，并完成用户所需的任何其他自定义任务。必须组织此过程以确保以正确的顺序执行任务，但同时尽可能快地执行。

#### 9.1.1 System V

System V 是经典的引导过程，自 1983 年以来一直在 Unix 和类 Unix 系统（如 Linux）中使用。它由一个小程序 init 组成，它设置诸如 login（通过 getty）之类的基本程序并运行一个脚本。该脚本通常名为 rc，控制一组附加脚本的执行，这些脚本执行初始化系统所需的任务。

init 程序由 `/etc/inittab` 文件控制，并组织成用户可以运行的运行级别：

* 0 — 停止
* 1 — 单用户模式
* 2 — 多用户且无网络
* 3 — 完整的多用户模式
* 4 — 用户可定义
* 5 — 带显示管理器的完整多用户模式
* 6 — 重新启动

通常的默认运行级别是 3 或 5。

优点：

* 制度完善，人尽皆知
* 易于定制

缺点：

* 启动速度可能较慢。中速基本 LFS 系统需要 8-12 秒，其中引导时间是从第一条内核消息到登录提示进行测量的。网络连接通常在登录提示后约 2 秒建立。
* 启动任务是串行处理的。上一点与有此关，任何进程（例如文件系统检查）的延迟都会延迟整个启动过程。
* 不直接支持控制组 (cgroups) 和每用户公平共享调度等高级功能。
* 添加脚本需要手动、静态的排序决策。

### 9.2 LFS-Bootscripts-20200818

LFS-Bootscripts 包含一组在启动/关闭时启动/停止 LFS 系统的脚本。以下部分描述了自定义引导过程所需的配置文件和过程。

解压软件包

```sh
(lfs chroot) root:/# cd /sources/
(lfs chroot) root:/sources# tar xf lfs-bootscripts-20200818.tar.xz 
(lfs chroot) root:/sources# cd lfs-bootscripts-20200818
```

安装软件包

```sh
(lfs chroot) root:/sources/lfs-bootscripts-20200818# make install
```

清除软件包

```sh
(lfs chroot) root:/sources/lfs-bootscripts-20200818# cd ..
(lfs chroot) root:/sources# rm -rf lfs-bootscripts-20200818
```

### 9.3 设备和模块处理概述

在第 8 章中，我们在构建 eudev 时安装了 udev 包。在我们深入了解其工作原理之前，先简要介绍一下以前处理设备的方法。

Linux 系统传统上使用静态设备创建方法，即在 /dev 下创建大量设备节点（有时实际上是数千个节点），而不管相应的硬件设备是否真的存在。这通常是通过 MAKEDEV 脚本完成的，该脚本包含对 mknod 程序的多次调用，以及世界上可能存在的每个可能设备的相关主设备号和次设备号。

现在，使用 udev 方法，只有那些被内核检测到的设备才会为它们创建设备节点。由于每次系统启动时都会创建这些设备节点，因此它们将存储在 devtmpfs 文件系统（完全驻留在系统内存中的虚拟文件系统）中。设备节点不需要太多空间，因此使用的内存可以忽略不计。

#### 9.3.1 历史

2000 年 2 月，一个名为 devfs 的新文件系统被合并到 2.3.46 内核中，并在 2.4 系列稳定内核期间可用。尽管它存在于内核源代码中，但这种动态创建设备的方法从未得到核心内核开发人员的压倒性支持。

devfs 采用的方法的**主要问题**是它处理设备检测、创建和命名的方式。另一个问题，即设备节点命名问题，可能是最关键的。人们普遍认为，如果允许设备名称可配置，那么设备命名策略应该由系统管理员决定，而不是由任何特定的开发人员强加给他们。devfs 文件系统还受到其设计固有的竞争条件的影响，如果不对内核进行实质性修改就无法修复。由于缺乏维护，它被标记为已弃用很长一段时间，最终于 2006 年 6 月从内核中删除。

随着不稳定的 2.5 内核树（后来作为 2.6 系列稳定内核发布）的开发，一个名为 sysfs 的新虚拟文件系统应运而生。sysfs 的工作是将系统硬件配置的视图导出到用户空间进程。通过这种用户空间可见的表示，开发用户空间替代 devfs 的可能性变得更加现实。

#### 9.3.2 Udev 实现

##### 9.3.2.1 Sysfs

sysfs 文件系统在上面简要提到过。人们可能想知道 sysfs 如何知道系统上存在的设备以及应该为它们使用哪些设备编号。已编译到内核中的驱动程序在内核检测到它们时直接向 sysfs（内部 devtmpfs）注册它们的对象。对于编译为模块的驱动程序，此注册将在加载模块时发生。一旦 sysfs 文件系统被挂载（在 /sys 上），驱动程序向 sysfs 注册的数据可供用户空间进程和 udevd 处理（包括对设备节点的修改）。

##### 9.3.2.2 设备节点的创建

设备文件由内核通过devtmpfs文件系统创建。任何希望注册设备节点devtmpfs的驱动程序都将通过 （通过驱动程序核心）来完成。当devtmpfs实例安装在 上时 /dev，设备节点最初将使用固定名称、权限和所有者创建。

不久之后，内核将向udevd发送一个 uevent 。基于在内的文件中指定的规则/etc/udev/rules.d，/lib/udev/rules.d和/run/udev/rules.d目录， udevd会将创建的符号链接附加到该设备节点，或改变它的权限，拥有者，或基，或修改内部 的udevd 该对象数据库条目（名称）。

这三个目录中的规则被编号，并且所有三个目录合并在一起。如果udevd找不到它正在创建的设备的规则，它将保留devtmpfs最初使用的权限和所有权。

##### 9.3.2.3 模块加载

编译为模块的设备驱动程序可能具有内置的别名。别名在modinfo 程序的输出中可见， 通常与模块支持的设备的总线特定标识符相关。例如，snd-fm801驱动程序支持供应商 ID 为 0x1319 和设备 ID 为 0x0801 的 PCI 设备，其别名为“ pci:v00001319d00000801sv*sd*bc04sc01i* ”。对于大多数设备，总线驱动程序导出将通过sysfs. 例如，/sys/bus/pci/devices/0000:00:0d.0/modalias 文件可能包含字符串"pci:v00001319d00000801sv00001319sd00001319bc04sc01i00"。udev 提供的默认规则会导致 udevd使用uevent 环境变量的内容（应该与sysfs中的文件内容相同）调用/sbin/modprobe，从而加载别名与此字符串匹配的所有模块通配符扩展后。 MODALIASmodalias

在这个例子中，这意味着，除了 snd-fm801 之外，如果有过时的（和不需要的）forte驱动程序，它就会被加载。请参阅下文，了解可以防止加载不需要的驱动程序的方法。

内核本身也能够按需加载网络协议、文件系统和 NLS 支持的模块。

##### 9.3.2.4 处理热插拔/动态设备

当您插入一个设备时，例如通用串行总线 (USB) MP3 播放器，内核会识别出该设备现在已连接并生成一个 uevent。这个 uevent 然后由udevd处理，如上所述。

#### 9.3.3 加载模块和创建设备的问题

自动创建设备节点时可能会出现一些问题。

##### 9.3.3.1 内核模块不会自动加载

如果模块具有特定于总线的别名并且总线驱动程序正确地将必要的别名导出到sysfs. 在其他情况下，应通过其他方式安排模块加载。在 Linux-5.8.3 中，udev 可以为 INPUT、IDE、PCI、USB、SCSI、SERIO 和 FireWire 设备加载正确编写的驱动程序。

要确定您需要的设备驱动程序是否具有对 udev 的必要支持，请使用模块名称作为参数运行modinfo。现在尝试定位下面的设备目录/sys/bus并检查那里是否有modalias 文件。

如果modalias文件存在于 中sysfs，则驱动程序支持该设备并可以直接与之对话，但没有别名，这是驱动程序中的错误。在没有 udev 帮助的情况下加载驱动程序，并期望稍后修复该问题。

如果modalias下相关目录中没有文件，则/sys/bus说明内核开发人员尚未为此总线类型添加模别名支持。对于 Linux-5.8.3，ISA 总线就是这种情况。预计此问题将在以后的内核版本中得到修复。

Udev 根本不打算加载诸如 snd-pcm-oss 之类的“包装器”驱动程序 和诸如loop 之类的非硬件驱动程序。

##### 9.3.3.2 内核模块不会自动加载，并且 udev 不打算加载它

如果“ wrapper ”模块只是增强了其他模块提供的功能（例如， snd-pcm-oss通过使声卡可用于 OSS 应用程序来增强snd-pcm的功能），请配置 modprobe在 udev 加载后加载包装器包裹的模块。为此，请在相应的 文件中添加“ softdep ”行。例如： /etc/modprobe.d/<filename>.conf

```sh
# (我不执行此命令)
softdep snd-pcm post: snd-pcm-oss
```

注意，“ softdep ”命令也允许 pre:依赖关系，或两者的混合物pre:和 post:依赖关系。有关“ softdep ”语法和功能的modprobe.d(5)更多信息，请参阅 手册页。

如果有问题的模块不是包装器并且本身很有用，请配置模块引导脚本以在系统引导时加载此模块。为此，请将模块名称添加到/etc/sysconfig/modules文件的单独行中。这也适用于包装器模块，但在这种情况下是次优的。

##### 9.3.3.3 Udev 加载了一些不需要的模块
要么不构建模块，要么/etc/modprobe.d/blacklist.conf像 下面示例中的forte模块那样将其列入黑名单 ：

```sh
# (我不执行此命令)
blacklist forte
```

仍然可以使用显式modprobe命令手动加载列入黑名单的模块。

##### 9.3.3.4 Udev 错误地创建了设备，或创建了错误的符号链接

如果规则意外匹配设备，通常会发生这种情况。例如，编写不当的规则可能会匹配 SCSI 磁盘（根据需要）和供应商对应的 SCSI 通用设备（错误地）。在udevadm info命令的帮助下，找出违规规则并使其更加具体。

##### 9.3.3.5 Udev 规则不可靠

这可能是之前问题的另一种表现。如果不是，并且您的规则使用sysfs属性，则可能是内核计时问题，将在以后的内核中修复。现在，您可以通过创建等待使用的sysfs属性并将其附加到/etc/udev/rules.d/10-wait_for_sysfs.rules 文件的规则来解决它（如果该文件不存在，则创建该文件）。如果您这样做，请通知 LFS 开发列表，这会有所帮助。

##### 9.3.3.6 udev 不创建设备

进一步的文本假设驱动程序是静态构建到内核中的或已经作为模块加载的，并且您已经检查过 udev 没有创建错误命名的设备。

如果内核驱动程序不将其数据导出到sysfs. 这在内核树之外的第三方驱动程序中最为常见。/lib/udev/devices使用适当的主要/次要编号创建静态设备节点（请参阅devices.txt内核文档中的文件或第三方驱动程序供应商提供的文档）。静态设备节点将被复制到/dev由udev的。

##### 9.3.3.7 重启后设备命名顺序随机变化

这是因为 udev 按照设计并行处理 uevent 和加载模块，因此顺序不可预测。这永远不会“固定”。您不应依赖内核设备名称的稳定性。相反，创建您自己的规则，根据设备的某些稳定属性（例如序列号或 udev 安装的各种 *_id 实用程序的输出）使用稳定名称创建符号链接。有关示例，请参见第 9.4 节“管理设备”和第 9.5 节“常规网络配置”。

#### 9.3.4. 帮助文档

以下站点提供了其他有用的帮助文档：

* devfs 的用户空间实现 http://www.kroah.com/linux/talks/ols_2003_udev_paper/Reprint-Kroah-Hartman-OLS2003.pdf

* sysfs文件系统 http://www.kernel.org/pub/linux/kernel/people/mochel/doc/papers/ols-2005/mochel.pdf

### 9.4 管理设备

#### 9.4.1 网络设备

默认情况下，Udev 根据固件/BIOS 数据或物理特性（如总线、插槽或 MAC 地址）命名网络设备。此命名约定的目的是确保网络设备的命名一致，而不是基于发现网卡的时间。比如一台有Intel和Realtek两块网卡的电脑，Intel制造的网卡可能变成eth0，Realtek的网卡变成eth1。在某些情况下，重新启动后，卡可能会以相反的方式重新编号。

在新的命名方案中，典型的网络设备名称将类似于 enp5s0 或 wlp3s0。如果不需要此命名约定，则可以实现传统命名方案或自定义方案。

##### 9.4.1.1 在内核命令行上禁用持久命名

使用 eth0、eth1 等的传统命名方案可以通过在内核命令行中添加 `net.ifnames=0` 来恢复。这最适合那些只有一个相同类型的以太网设备的系统。笔记本电脑通常有多个名为 eth0 和 wlan0 的以太网连接，也是这种方法的候选者。命令行在 GRUB 配置文件中传递。请参阅第 10.4.4 节，“创建 GRUB 配置文件”。

##### 9.4.1.2 创建自定义 Udev 规则

命名方案可以通过创建自定义 udev 规则来自定义。已包含生成初始规则的脚本。通过以下命令生成这些规则：

```sh
(lfs chroot) root:/sources# bash /lib/udev/init-net-rules.sh
```

现在，检查 `/etc/udev/rules.d/70-persistent-net.rules` 文件，找出分配给哪个网络设备的名称：

```sh
(lfs chroot) root:/sources# cat /etc/udev/rules.d/70-persistent-net.rules
cat: /etc/udev/rules.d/70-persistent-net.rules: No such file or directory
# 我这里没有自动生成该文件
```

注意：在某些情况下，例如手动将 MAC 地址分配给网卡或在 Qemu 或 Xen 等虚拟环境中时，**可能不会生成网络规则文件**，因为地址分配不一致。在这些情况下，不能使用此方法。

该文件以注释块开头，后跟每个 NIC 的两行。每个 NIC 的第一行是带注释的描述，显示其硬件 ID（例如，其 PCI 供应商和设备 ID，如果是 PCI 卡），以及括号中的驱动程序（如果可以找到驱动程序）。硬件 ID 和驱动程序都不用于确定为接口指定哪个名称；此信息仅供参考。第二行是 udev 规则，它匹配这个 NIC 并实际为其分配一个名称。

所有 udev 规则都由几个键组成，用逗号和可选的空格分隔。该规则的键和对每个键的解释如下：

* `SUBSYSTEM=="net"` - 这告诉 udev 忽略不是网卡的设备。
* `ACTION=="add"` - 这告诉 udev 对于不是 add 的 uevent 忽略此规则（“remove”和“change” uevents 也会发生，但不需要重命名网络接口）。
* `DRIVERS=="?*"` - 存在这样 udev 将忽略 VLAN 或桥接子接口（因为这些子接口没有驱动程序）。这些子接口将被跳过，因为分配的名称会与其父设备发生冲突。
* `ATTR{address}` - 该键的值是 NIC 的 MAC 地址。
* `ATTR{type}=="1"` - 这确保规则仅在某些创建多个虚拟接口的无线驱动程序的情况下匹配主接口。跳过辅助接口的原因与跳过 VLAN 和网桥子接口的原因相同：否则会发生名称冲突。
* `NAME` - 该键的值是 udev 将分配给该接口的名称。

`NAME` 的值是重要的部分。在继续之前，请确保您知道已为每个网卡分配了哪个名称，并确保在下面创建配置文件时使用该 NAME 值。

可能是由于我没有网卡设备，所以才出现的问题，**这个问题是可以忽略！**。

在接下来的第9.5章节中，针对此问题进行了说明，现在您可以继续执行下一小节的内容：

我**自作聪明地**参考了 https://www.cnblogs.com/coolYuan/p/11077614.html 的解决方案，实际上我不需要这样做：

```sh
# 以下代码内容不需要执行！
(lfs chroot) root:/sources# cat /etc/udev/rules.d/70-persistent-net.rules
cat: /etc/udev/rules.d/70-persistent-net.rules: No such file or directory
(lfs chroot) root:/sources# /lib/udev/write_net_rules
missing $INTERFACE
(lfs chroot) root:/sources# export INTERFACE="eth0"
(lfs chroot) root:/sources# /lib/udev/write_net_rules
missing valid match
(lfs chroot) root:/sources# export MATCHADDR="23:33:33:33:33:65" # 我瞎编了一个MAC地址
(lfs chroot) root:/sources# /lib/udev/write_net_rules
(lfs chroot) root:/sources# cat /etc/udev/rules.d/70-persistent-net.rules
# This file was automatically generated by the /lib/udev/write_net_rules
# program, run by the persistent-net-generator.rules rules file.
#
# You can modify it, as long as you keep each rule on a single
# line, and change only the value of the NAME= key.

SUBSYSTEM=="net", ACTION=="add", DRIVERS=="?*", ATTR{address}=="23:33:33:33:33:65", NAME="eth0"
(lfs chroot) root:/sources# 
```

#### 9.4.2. CD-ROM 链接

您可能希望稍后安装的某些软件（例如，各种媒体播放器）需要 `/dev/cdrom` 和 `/dev/dvd` 符号链接存在，并指向 CD-ROM 或 DVD-ROM 设备。将这些符号链接的引用放入 `/etc/fstab` 可能会很方便。Udev 附带了一个脚本，该脚本将生成规则文件来为您创建这些符号链接，具体取决于每个设备的功能，但您需要决定您希望脚本使用两种操作模式中的哪一种。

首先，脚本可以在 `by-path` 模式下运行（默认用于 USB 和 FireWire 设备），它创建的规则取决于 CD 或 DVD 设备的物理路径。其次，它可以在 `by-id` 模式（IDE 和 SCSI 设备的默认设置）下运行，它创建的规则取决于存储在 CD 或 DVD 设备本身上的标识字符串。路径由 udev 的 `path_id` 脚本确定，标识字符串由其 `ata_id` 或 `scsi_id` 程序从硬件中读取，具体取决于您拥有的设备类型。

每种方法都有其优点；正确的使用方法取决于可能发生的设备更改类型。如果您希望设备的物理路径（即它插入的端口和/或插槽）发生变化，例如，因为您计划将驱动器移动到不同的 IDE 端口或不同的 USB 连接器，那么您应该使用 `by-id` 模式。另一方面，如果您预计设备的标识会发生变化，例如因为它可能会死机，如果您将其替换为具有相同功能且插入相同连接器的不同设备，那么您应该使用 `by-path` 模式。

如果您的驱动器可以进行任一类型的更改，请根据您希望更频繁发生的更改类型选择一种模式。

重要：外部设备（例如，USB 连接的 CD 驱动器）不应使用 `by-path` ，因为每次将设备插入新的外部端口时，其物理路径都会发生变化。如果你写udev规则 `by-path` 来识别它们，所有外部连接的设备都会有这个问题；问题不仅限于 CD 和 DVD 驱动器。

如果您想查看 udev 脚本将使用的值，那么对于相应的 CD-ROM 设备，在 /sys 下找到相应的目录（例如，这可以是 /sys/block/hdd）并运行类似于下列的：

```sh
# 以下代码内容不需要执行！
(lfs chroot) root:/sources# udevadm test /sys/block/hdd
calling: test
version 3.2.9
This program is for debugging only, it does not run any program
specified by a RUN key. It may show incorrect results, because
some values may be different, or not available at a simulation run.

=== trie on-disk ===
tool version:          3
file size:         8270736 bytes
header size             80 bytes
strings            2110312 bytes
nodes              6160344 bytes
Load module index
timestamp of '/etc/udev/rules.d' changed
Reading rules file: /lib/udev/rules.d/50-udev-default.rules
Reading rules file: /etc/udev/rules.d/55-lfs.rules
Reading rules file: /lib/udev/rules.d/60-block.rules
Reading rules file: /lib/udev/rules.d/60-cdrom_id.rules
Reading rules file: /lib/udev/rules.d/60-drm.rules
Reading rules file: /lib/udev/rules.d/60-evdev.rules
Reading rules file: /lib/udev/rules.d/60-input-id.rules
Reading rules file: /lib/udev/rules.d/60-persistent-alsa.rules
Reading rules file: /lib/udev/rules.d/60-persistent-input.rules
Reading rules file: /lib/udev/rules.d/60-persistent-storage-tape.rules
Reading rules file: /lib/udev/rules.d/60-persistent-storage.rules
Reading rules file: /lib/udev/rules.d/60-persistent-v4l.rules
Reading rules file: /lib/udev/rules.d/60-sensor.rules
Reading rules file: /lib/udev/rules.d/60-serial.rules
Reading rules file: /lib/udev/rules.d/64-btrfs.rules
Reading rules file: /lib/udev/rules.d/70-joystick.rules
Reading rules file: /lib/udev/rules.d/70-mouse.rules
Reading rules file: /etc/udev/rules.d/70-persistent-net.rules
Reading rules file: /lib/udev/rules.d/70-touchpad.rules
Reading rules file: /lib/udev/rules.d/75-net-description.rules
Reading rules file: /lib/udev/rules.d/75-probe_mtd.rules
Reading rules file: /lib/udev/rules.d/78-sound-card.rules
Reading rules file: /lib/udev/rules.d/80-drivers.rules
Reading rules file: /lib/udev/rules.d/80-net-name-slot.rules
Reading rules file: /etc/udev/rules.d/81-cdrom.rules
Reading rules file: /etc/udev/rules.d/83-cdrom-symlinks.rules
Reading rules file: /lib/udev/rules.d/96-e2scrub.rules
rules contain 24576 bytes tokens (2048 * 12 bytes), 8710 bytes strings
1140 strings (14949 bytes), 712 de-duplicated (6668 bytes), 429 trie nodes used
unable to open device '/sys/block/hdd'
Unload module index
```

查看包含各种 *_id 程序输出的行。`by-id` 模式将使用 ID_SERIAL 值，如果它存在且不为空；否则它将使用 ID_MODEL 和 ID_REVISION 的组合。 `by-path` 模式将使用 ID_PATH 值。

如果默认模式不适合你的情况，那么可以对 `/etc/udev/rules.d/83-cdrom-symlinks.rules` 文件进行如下修改，如下（其中模式是 `by-id` 或 `by-id`）：

```sh
# (我不运行此命令)
sed -e 's/"write_cd_rules"/"write_cd_rules mode"/' \
    -i /etc/udev/rules.d/83-cdrom-symlinks.rules
```

请注意，此时没有必要创建规则文件或符号链接，因为您已将宿主机的 /dev 目录绑定挂载到 LFS 系统中，并且我们假设符号链接存在于宿主机上。规则和符号链接将在您第一次启动 LFS 系统时创建。

但是，如果您有多个 CD-ROM 设备，那么当时生成的符号链接可能指向不同的设备，而不是它们指向您宿主机上的设备，因为设备不会以可预测的顺序被发现。首次启动 LFS 系统时创建的分配将是稳定的，因此只有当您需要两个系统上的符号链接指向同一设备时，这才是一个问题。如果需要，请在启动后检查（并可能编辑）生成的 `/etc/udev/rules.d/70-persistent-cd.rules` 文件，以确保分配的符号链接符合您的需要。

#### 9.4.3 处理重复设备

如第 9.3 节“设备和模块处理概述”所述，具有相同功能的设备出现在 `/dev` 中的顺序基本上是随机的。如果您有 USB 网络摄像头和电视调谐器，有时 `/dev/video0` 指的是摄像头，而 `/dev/video1` 指的是调谐器，有时重新启动后顺序会发生变化。对于除声卡和网卡之外的所有硬件类别，这可以通过为自定义持久符号链接创建 udev 规则来解决。网卡的情况在第 9.5 节“一般网络配置”中单独介绍，声卡配置可以在 BLFS 中找到。

对于可能出现此问题的每个设备（即使该问题在您当前的 Linux 发行版中不存在），在 `/sys/class` 或 `/sys/block` 下找到对应的目录。对于视频设备，这可能是 `/sys/class/video4linux/videoX`。对于视频设备，这可能是 `/sys/class/video4linux/videoX`。找出唯一标识设备的属性（通常，供应商和产品 ID 和/或序列号有效）：

```sh
# (我不运行此命令)
udevadm info -a -p /sys/class/video4linux/video0
```

然后编写创建符号链接的规则，例如：

```sh
# (我不运行此命令)
cat > /etc/udev/rules.d/83-duplicate_devs.rules << "EOF"

# Persistent symlinks for webcam and tuner
KERNEL=="video*", ATTRS{idProduct}=="1910", ATTRS{idVendor}=="0d81", SYMLINK+="webcam"
KERNEL=="video*", ATTRS{device}=="0x036f",  ATTRS{vendor}=="0x109e", SYMLINK+="tvtuner"

EOF
```

结果是 `/dev/video0` 和 `/dev/video1` 设备仍然随机引用调谐器和网络摄像头（因此永远不应直接使用），但有符号链接 `/dev/tvtuner` 和 `/dev/webcam` 总是指向到正确的设备。

### 9.5 通用网络设置

#### 9.5.1 创建网络接口配置文件

网络脚本启动和关闭哪些接口通常取决于 `/etc/sysconfig/` 中的文件。这个目录应该包含每个要配置的接口的文件，例如 `ifconfig.xyz`，其中 `xyz` 应该描述网卡。接口名称（例如 eth0）通常是合适的。此文件中包含此接口的属性，例如其 IP 地址、子网掩码等。文件名的主干必须是 ifconfig。

如果没有使用上一节(9.4 章节)中的过程，udev 将根据系统物理特性（例如 enp2s1）分配网卡接口名称。如果您不确定您的接口名称是什么，您可以在启动系统后运行 `ip link` 或 `ls /sys/class/net`。

以下命令为具有静态 IP 地址的 eth0 设备创建示例文件：

```sh
# 其中的一些配置需要根据你的实际情况进行修改
# 运行此脚本：
cd /etc/sysconfig/
cat > ifconfig.eth0 << "EOF"
ONBOOT=yes
IFACE=eth0
SERVICE=ipv4-static
IP=192.168.3.66
GATEWAY=192.168.3.1
PREFIX=24
BROADCAST=192.168.3.255
EOF
```

每条语句等于号后面的值必须在每个文件中更改以匹配正确的设置。

* 如果 `ONBOOT` 变量设置为 `yes`，System V 网络脚本将在系统引导过程中启动网络接口卡 (NIC)。如果设置为 `yes` 以外的任何值，网络脚本将忽略 NIC，并且不会自动启动。可以使用 `ifup` 和 `ifdown` 命令手动启动或停止接口。
* `IFACE` 变量定义了接口名称，例如 eth0。所有网络设备配置文件都需要它。文件扩展名必须与此值匹配。所有网络设备配置文件都需要它，并且文件扩展名必须与此值匹配。
* `SERVICE` 变量定义了用于获取 IP 地址的方法。LFS-Bootscripts 包具有模块化的 IP 分配格式，在 `/lib/services/` 目录中创建附加文件允许使用其他 IP 分配方法。这通常用于动态主机配置协议 (DHCP)，它在 BLFS 书中有介绍。
* `GATEWAY` 变量应包含默认网关 IP 地址（如果存在）。如果没有，则完全注释掉变量。
* `PREFIX` 变量包含子网中使用的位数。IP 地址中的每个八位字节为 8 位。如果子网的网络掩码是 255.255.255.0，则它使用前三个八位字节（24 位）来指定网络号。如果网络掩码是 255.255.255.240，它将使用前 28 位。长度超过 24 位的前缀通常由 DSL 和基于电缆的 Internet 服务提供商 (ISP) 使用。在本例中（PREFIX=24），网络掩码是 255.255.255.0。根据您的特定子网调整 PREFIX 变量。如果省略，PREFIX 默认为 24。

有关更多信息，请参阅 `ifup` 手册页。

#### 9.5.2 创建 /etc/resolv.conf 文件

系统将需要一些获取域名服务 (DNS) 名称解析的方法，以将 Internet 域名解析为 IP 地址，反之亦然。这最好通过将 DNS 服务器的 IP 地址（可从 ISP 或网络管理员获得）放入 `/etc/resolv.conf` 中来实现。通过运行以下命令创建文件：

```sh
# (不要执行以下命令！)
cat > /etc/resolv.conf << "EOF"
# Begin /etc/resolv.conf

domain <Your Domain Name>
nameserver <IP address of your primary nameserver>
nameserver <IP address of your secondary nameserver>

# End /etc/resolv.conf
EOF
```

其中， `domain` 段可以省略或替换为 `search` 段。有关详细信息，请参阅 resolv.conf 的手册页。

将 `<IP address of your primary nameserver>` 替换为最适合设置的 DNS 的 IP 地址。这通常会有多个条目（要求需要辅助服务器以提供回退功能）。如果您只需要或想要一台 DNS 服务器，请从文件中删除第二个名称服务器行。 IP 地址也可能是本地网络上的路由器。

根据以上信息，执行下面的命令：

```sh
cat > /etc/resolv.conf << "EOF"
# Begin /etc/resolv.conf

nameserver 192.168.3.1

# End /etc/resolv.conf
EOF
```

#### 9.5.3 配置系统主机名

在引导过程中，文件 `/etc/hostname` 用于建立系统的主机名。

创建 `/etc/hostname` 文件并通过运行以下命令输入主机名：

```sh
echo "lfs-system" > /etc/hostname
```

`lfs-system` 需要替换为计算机的名称。不要在此处输入完全限定域名 (FQDN)。该信息放在 `/etc/hosts` 文件中。

#### 9.5.4 自定义 /etc/hosts 文件

确定 IP 地址、完全限定域名 (FQDN) 和可能在 /etc/hosts 文件中使用的别名。语法是：

```sh
IP_address myhost.example.org aliases
```

除非该计算机可以从 Internet 访问 (即拥有一个注册域名，并分配了一个有效的 IP 地址段 —— 多数用户没有分配有效 IP)，确认使用的 IP 地址属于私网 IP 范围。有效的范围是：

```
私网地址范围                          公共前缀长度
10.0.0.1 - 10.255.255.254            8
172.x.0.1 - 172.x.255.254            16
192.168.y.1 - 192.168.y.254          24
```

`x` 可以是 16-31 范围内的任何数字。 `y` 可以是 0-255 范围内的任何数字。

有效的私有 IP 地址可能是 `192.168.1.1`。此 IP 的有效 `FQDN` 可以是 `lfs.example.org`。

即使不使用网卡，仍然需要有效的 `FQDN` 。这是某些程序正确运行所必需的。

通过运行创建 `/etc/hosts` 文件：

```sh
# 不要执行以下指令：
cat > /etc/hosts << "EOF"
# Begin /etc/hosts

127.0.0.1 localhost.localdomain localhost
127.0.1.1 <FQDN> <HOSTNAME>
<192.168.1.1> <FQDN> <HOSTNAME> [alias1] [alias2 ...]
::1       localhost ip6-localhost ip6-loopback
ff02::1   ip6-allnodes
ff02::2   ip6-allrouters

# End /etc/hosts
EOF
```

`<192.168.1.1>`、`<FQDN>` 和 `<HOSTNAME>` 值需要根据特定用途或要求进行更改（如果网络/系统管理员分配了 IP 地址并且机器将连接到现有网络）。可以省略可选的别名。

根据以上说明，我执行以下代码：

```sh
cat > /etc/hosts << "EOF"
# Begin /etc/hosts

127.0.0.1 localhost.localdomain localhost
127.0.1.1 lfs-system

::1       ip6-localhost ip6-loopback
ff02::1   ip6-allnodes
ff02::2   ip6-allrouters

# End /etc/hosts
EOF
```

### 9.6 System V 引导脚本的使用和配置

#### 9.6.1 System V 引导脚本如何工作

Linux 使用一个名为 SysVinit 的特殊引导工具，**它基于运行级别的概念**。不同的系统可能会有很大的不同，所以不能假设由于是在一个特定的 Linux 发行版中工作，因而它们在 LFS 中也应该同样工作。LFS 有自己的做事方式，但它尊重普遍接受的标准。

SysVinit（从现在开始称为 `init` ）使用运行级别方案工作。有七个（编号为 0 到 6）运行级别（实际上，运行级别还有更多，但它们是针对特殊情况的，一般不使用。有关更多详细信息，请参阅 init(8)，其中每一项都对应于计算机在启动时应该执行的操作。**默认运行级别为 3**。以下是对不同运行级别的描述：

* 0: 停止计算机
* 1: 单用户模式
* 2: 没有网络的多用户模式
* 3: 带网络的多用户模式
* 4: 保留用于定制，其他与 3 相同
* 5: 与 4 相同，它通常用于 GUI 登录（如 X 的 xdm 或 KDE 的 kdm）
* 6: 重新启动计算机

#### 9.6.2 配置 Sysvinit

在内核初始化期间，运行的第一个程序要么在命令行中指定，要么默认为 init。该程序读取初始化文件 `/etc/inittab`。使用以下命令创建此文件：

```sh
cat > /etc/inittab << "EOF"
# Begin /etc/inittab

id:3:initdefault:

si::sysinit:/etc/rc.d/init.d/rc S

l0:0:wait:/etc/rc.d/init.d/rc 0
l1:S1:wait:/etc/rc.d/init.d/rc 1
l2:2:wait:/etc/rc.d/init.d/rc 2
l3:3:wait:/etc/rc.d/init.d/rc 3
l4:4:wait:/etc/rc.d/init.d/rc 4
l5:5:wait:/etc/rc.d/init.d/rc 5
l6:6:wait:/etc/rc.d/init.d/rc 6

ca:12345:ctrlaltdel:/sbin/shutdown -t1 -a -r now

su:S016:once:/sbin/sulogin

1:2345:respawn:/sbin/agetty --noclear tty1 9600
2:2345:respawn:/sbin/agetty tty2 9600
3:2345:respawn:/sbin/agetty tty3 9600
4:2345:respawn:/sbin/agetty tty4 9600
5:2345:respawn:/sbin/agetty tty5 9600
6:2345:respawn:/sbin/agetty tty6 9600

# End /etc/inittab
EOF
```

此初始化文件的说明在 inittab 的手册页中。对于 LFS，运行的关键命令是 rc。上面的初始化文件将指示 rc 运行 `/etc/rc.d/rcS.d` 目录中所有以 S 开头的脚本后跟 `/etc/rc.d/rc?.d` 目录中所有以 S 开头的脚本，其中问号由 initdefault 值指定。

为方便调试，函数脚本还将所有输出记录到 `/run/var/bootlog`。

#### 9.6.3 Udev 引导脚本

/etc/rc.d/init.d/udev initscript 启动 udevd，触发任何已经由内核创建的“冷插拔”设备并等待任何规则完成。该脚本还会从 /sbin/hotplug 的默认设置中取消设置 uevent 处理程序。

#### 9.6.4 配置系统时钟

setclock 脚本从硬件时钟（也称为 BIOS 或互补金属氧化物半导体 (CMOS) 时钟）读取时间。如果硬件时钟设置为 UTC，此脚本将使用 /etc/localtime 文件（告诉 hwclock 程序使用哪个时区）将硬件时钟的时间转换为本地时间。无法检测硬件时钟是否设置为UTC，因此需要手动配置。

当内核在启动时检测到硬件能力时，setclock 程序通过 udev 运行。也可以手动运行并带有停止参数，将系统时间存储到 CMOS 时钟。

如果您不记得硬件时钟是否设置为 UTC，请通过运行 `hwclock --localtime --show` 命令找出。这将根据硬件时钟显示当前时间。如果此时间与您的手表所说的相符，则硬件时钟将设置为本地时间。如果 hwclock 的输出不是本地时间，则很可能将其设置为 UTC 时间。通过在 hwclock 显示的时间上加上或减去适当的时区小时数来验证这一点。例如，如果您当前处于 MST 时区（也称为 GMT -0700），则将本地时间添加 7 小时。

```sh
(lfs chroot) root:/# hwclock --localtime --show
2021-12-23 09:22:48.128431+08:00
# 以上说明 UTC 时间已经被设置
```

如果硬件时钟未设置为 UTC 时间，请将以下 `UTC` 变量的值更改为 `0`（零）值。

通过运行以下命令创建一个新文件 `/etc/sysconfig/clock`：

```sh
cat > /etc/sysconfig/clock << "EOF"
# Begin /etc/sysconfig/clock

UTC=1

# Set this to any options you might need to give to hwclock,
# such as machine hardware clock type for Alphas.
CLOCKPARAMS=

# End /etc/sysconfig/clock
EOF
```

以下链接提供了解释如何处理 LFS 上时间的一个很好的提示。它解释了诸如时区、UTC 和 TZ 环境变量等问题。http://www.linuxfromscratch.org/hints/downloads/files/time.txt

注意： `CLOCKPARAMS` 和 `UTC` 参数也可以在 `/etc/sysconfig/rc.site` 文件中设置。

#### 9.6.5 配置 Linux 控制台

本节讨论如何配置用于设置键盘映射、控制台字体和控制台内核日志级别的控制台引导脚本。如果不使用非 ASCII 字符（例如，版权符号、英镑符号和欧元符号）并且键盘是美国键盘，则可以跳过本节的大部分内容。如果没有配置文件（或 中的等效设置rc.site）， 控制台 引导脚本将什么也不做。

在控制台 脚本读取/etc/sysconfig/console的配置信息文件。决定使用哪种键盘映射和屏幕字体。各种特定于语言的 HOWTO 也可以帮助解决这个问题，请参阅http://www.tldp.org/HOWTO/HOWTO-INDEX/other-lang.html。如果仍有疑问，请在/usr/share/keymaps和/usr/share/consolefonts目录中查找有效的键盘映射和屏幕字体。阅读loadkeys(1)和setfont(8)手册，来确定这些程序的正确参数。

该/etc/sysconfig/console文件应包含以下形式的行：VARIABLE="value"。识别以下变量：

* 日志级别

此变量指定发送到控制台的内核消息的日志级别，如dmesg -n所设置。有效级别从“1”（无消息）到“8”。默认级别为“7”。

* 键盘映射

此变量指定loadkeys 程序的参数 ，通常是要加载的键映射的名称，例如 “ it ”。如果未设置此变量，则引导脚本将不会运行loadkeys程序，将使用默认的内核键盘映射。请注意，一些键映射具有多个具有相同名称的版本（在 qwerty/ 和 qwertz/ 中为 cz 及其变体，在 olpc/ 和 qwerty/ 中为 es，在 fgGIod/ 和 qwerty/ 中为 trf）。在这些情况下，还应指定父目录（例如 qwerty/es）以确保加载正确的键盘映射。

* KEYMAP_CORRECTIONS

这个（很少使用）变量指定了第二次调用loadkeys程序的参数。如果库存键盘映射不完全令人满意并且必须进行小幅调整，这将很有用。例如，要将欧元符号包含到通常没有它的键映射中，请将此变量设置为 “ euro2 ”。

* 字体

此变量指定setfont 程序的参数 。通常，这包括字体名称 “ -m ”和要加载的应用程序字符映射的名称。例如，为了将“ lat1-16 ”字体与“ 8859-1 ”应用程序字符映射（在美国适用）一起加载，将此变量设置为“ lat1-16 -m 8859-1 ”. 在 UTF-8 模式下，内核使用应用程序字符映射将键盘映射中的组合 8 位键代码转换为 UTF-8，因此“-m”参数的参数应设置为组合的编码键映射中的键代码。

* 统一编码

将此变量设置为“ 1 ”、“ yes ”或“ true ”，以便将控制台置于 UTF-8 模式。这在基于 UTF-8 的语言环境中很有用，否则有害。

* LEGACY_CHARSET

对于许多键盘布局，Kbd 包中没有库存的 Unicode 键盘映射。该控制台初始化脚本不会如果该变量设置为可用非UTF-8键盘映射的编码可用键盘映射转换为UTF-8的飞行。

一些例子：

对于非 Unicode 设置，通常只需要 KEYMAP 和 FONT 变量。例如，对于波兰语设置，可以使用：

```sh
# 我不执行以下代码：
cat > /etc/sysconfig/console << "EOF"
# Begin /etc/sysconfig/console

KEYMAP="pl2"
FONT="lat2a-16 -m 8859-2"

# End /etc/sysconfig/console
EOF
```

如上所述，有时需要稍微调整股票键盘映射。以下示例将欧元符号添加到德语键盘映射中：

```sh
# 我不执行以下代码：
cat > /etc/sysconfig/console << "EOF"
# Begin /etc/sysconfig/console

KEYMAP="de-latin1"
KEYMAP_CORRECTIONS="euro2"
FONT="lat0-16 -m 8859-15"
UNICODE="1"

# End /etc/sysconfig/console
EOF
```

以下是一个支持 Unicode 的保加利亚语示例，其中存在一个常用的 UTF-8 键盘映射：

```sh
# 我不执行以下代码：
cat > /etc/sysconfig/console << "EOF"
# Begin /etc/sysconfig/console

UNICODE="1"
KEYMAP="bg_bds-utf8"
FONT="LatArCyrHeb-16"

# End /etc/sysconfig/console
EOF
```

由于在前面的示例中使用了 512 字形的 LatArCyrHeb-16 字体，除非使用帧缓冲区，否则 Linux 控制台上不再提供明亮的颜色。如果一个人想要在没有帧缓冲区的情况下拥有明亮的颜色并且可以在没有不属于他的语言的字符的情况下生活，仍然可以使用特定于语言的 256 字形字体，如下图所示：

```sh
# 我不执行以下代码：
cat > /etc/sysconfig/console << "EOF"
# Begin /etc/sysconfig/console

UNICODE="1"
KEYMAP="bg_bds-utf8"
FONT="cyr-sun16"

# End /etc/sysconfig/console
EOF
```

以下示例说明了从 ISO-8859-15 到 UTF-8 的键映射自动转换以及在 Unicode 模式下启用死键：

```sh
# 我不执行以下代码：
cat > /etc/sysconfig/console << "EOF"
# Begin /etc/sysconfig/console

UNICODE="1"
KEYMAP="de-latin1"
KEYMAP_CORRECTIONS="euro2"
LEGACY_CHARSET="iso-8859-15"
FONT="LatArCyrHeb-16 -m 8859-15"

# End /etc/sysconfig/console
EOF
```

**对于中文、日文、韩文和其他一些语言，Linux 控制台无法配置为显示所需的字符。需要这些语言的用户应该安装 X Window 系统、覆盖必要字符范围的字体和正确的输入法（例如，SCIM，支持多种语言）。**

#### 9.6.6 在启动时创建文件

有时，希望在启动时创建文件。例如，`/tmp/.ICE-unix` 经常需要目录。这可以通过在 `/etc/sysconfig/createfiles` 配置脚本中创建一个条目来完成。该文件的格式嵌入在默认配置文件的注释中。

#### 9.6.7 配置 sysklogd 脚本

该 sysklogd 脚本调用 syslogd 程序作为 System V 初始化的一部分。默认情况下，该 `-m 0` 选项会关闭 syslogd 每 20 分钟写入日志文件的定期时间戳标记。如果要打开此周期性时间戳标记，请编辑`/etc/sysconfig/rc.site` 变量 `SYSKLOGD_PARMS` 并将其定义为所需值。例如，要删除所有参数，请将变量设置为空值：

```sh
# 执行以下命令：
(lfs chroot) root:/# SYSKLOGD_PARMS=
```

#### 9.6.8 rc.site 文件

可选 `/etc/sysconfig/rc.site` 文件包含为每个 System V 引导脚本自动设置的设置。它可以交替设置在指定的值 `hostname`，`console` 以及 `clock` 在文件 `/etc/sysconfig/` 目录。如果相关变量同时存在于这些单独的文件和 `rc.site` 中，则脚本特定文件中的值具有优先权。

`rc.site` 还包含可以自定义引导过程其他方面的参数。设置 `IPROMPT` 变量将启用引导脚本的选择性运行。其他选项在文件注释中描述。该文件默认配置如下：

```sh
# rc.site
# Optional parameters for boot scripts.

# Distro Information
# These values, if specified here, override the defaults
#DISTRO="Linux From Scratch" # The distro name
#DISTRO_CONTACT="lfs-dev@linuxfromscratch.org" # Bug report address
#DISTRO_MINI="LFS" # Short name used in filenames for distro config

# Define custom colors used in messages printed to the screen

# Please consult `man console_codes` for more information
# under the "ECMA-48 Set Graphics Rendition" section
#
# Warning: when switching from a 8bit to a 9bit font,
# the linux console will reinterpret the bold (1;) to
# the top 256 glyphs of the 9bit font.  This does
# not affect framebuffer consoles

# These values, if specified here, override the defaults
#BRACKET="\\033[1;34m" # Blue
#FAILURE="\\033[1;31m" # Red
#INFO="\\033[1;36m"    # Cyan
#NORMAL="\\033[0;39m"  # Grey
#SUCCESS="\\033[1;32m" # Green
#WARNING="\\033[1;33m" # Yellow

# Use a colored prefix
# These values, if specified here, override the defaults
#BMPREFIX="      "
#SUCCESS_PREFIX="${SUCCESS}  *  ${NORMAL} "
#FAILURE_PREFIX="${FAILURE}*****${NORMAL} "
#WARNING_PREFIX="${WARNING} *** ${NORMAL} "

# Manually seet the right edge of message output (characters)
# Useful when resetting console font during boot to override
# automatic screen width detection
#COLUMNS=120

# Interactive startup
#IPROMPT="yes" # Whether to display the interactive boot prompt
#itime="3"    # The amount of time (in seconds) to display the prompt

# The total length of the distro welcome string, without escape codes
#wlen=$(echo "Welcome to ${DISTRO}" | wc -c )
#welcome_message="Welcome to ${INFO}${DISTRO}${NORMAL}"

# The total length of the interactive string, without escape codes
#ilen=$(echo "Press 'I' to enter interactive startup" | wc -c )
#i_message="Press '${FAILURE}I${NORMAL}' to enter interactive startup"

# Set scripts to skip the file system check on reboot
#FASTBOOT=yes

# Skip reading from the console
#HEADLESS=yes

# Write out fsck progress if yes
#VERBOSE_FSCK=no

# Speed up boot without waiting for settle in udev
#OMIT_UDEV_SETTLE=y

# Speed up boot without waiting for settle in udev_retry
#OMIT_UDEV_RETRY_SETTLE=yes

# Skip cleaning /tmp if yes
#SKIPTMPCLEAN=no

# For setclock
#UTC=1
#CLOCKPARAMS=

# For consolelog (Note that the default, 7=debug, is noisy)
#LOGLEVEL=7

# For network
#HOSTNAME=mylfs

# Delay between TERM and KILL signals at shutdown
#KILLDELAY=3

# Optional sysklogd parameters
#SYSKLOGD_PARMS="-m 0"

# Console parameters
#UNICODE=1
#KEYMAP="de-latin1"
#KEYMAP_CORRECTIONS="euro2"
#FONT="lat0-16 -m 8859-15"
#LEGACY_CHARSET=
```

我对该文件的默认版本进行了一些修改：

```sh
# Distro Information
# These values, if specified here, override the defaults
#DISTRO="Linux From Scratch build by ZHJ0125 v0.1.0" # The distro name
#DISTRO_CONTACT="shandonghoujin@163.com" # Bug report address
#DISTRO_MINI="LFS" # Short name used in filenames for distro config
# Define custom colors used in messages printed to the screen
```

### 9.7 Bash Shell 启动文件

shell 程序 `/bin/bash`（以下简称 `shell`）使用一组启动文件来帮助创建一个运行环境。每个文件都有特定的用途，可能会以不同的方式影响登录和交互环境。/etc 目录中的文件提供全局设置。如果主目录中存在等效文件，则它可能会覆盖全局设置。

通过读取文件 ，使用 `/bin/login` 成功登录后启动交互式登录 `shell` `/etc/passwd`。交互式非登录 shell 在命令行（例如`[prompt]$/bin/bash`）处启动。当 shell 脚本运行时，通常会出现一个非交互式 shell。它是非交互式的，因为它正在处理脚本而不是在命令之间等待用户输入。

有关更多信息，请参阅 Bash 启动文件和交互式 Shell 部分下的 info bash。

**当 shell 作为交互式登录时 `/etc/profile`，`~/.bash_profile` 将读取这些文件。**

`/etc/profile` 文件中设置了一些本地语言支持所需的环境变量。正确设置它们会导致：

* 翻译成母语的程序输出

* 将字符正确分类为字母、数字和其他类别。这是bash在非英语语言环境中正确接受命令行中的非 ASCII 字符 所必需的
* 国家的正确字母排序顺序
* 合适的默认纸张尺寸
* 货币、时间和日期值的正确格式

将 `<ll>` 下面替换为所需语言的两字母代码（例如 `en`）和 `<CC>` 相应国家/地区的两字母代码（例如 `GB`）。 `<charmap>` 应替换为您选择的语言环境的规范 charmap。 也可能存在诸如 `@euro` 之类的可选修饰符。

可以通过运行以下命令获得 Glibc 支持的所有语言环境的列表：

```sh
(lfs chroot) root:/# locale -a
```

Charmap 可以有多个别名，例如， `ISO-8859-1` 也称为 `iso8859-1` 和 `iso88591` 。某些应用程序无法正确处理各种同义词（例如，要求将 `UTF-8` 写为 `UTF-8`，而不是 `utf8`），因此在大多数情况下为特定语言环境选择规范名称是最安全的。要确定规范名称，请运行以下命令，其中 `<locale name>` 是 `locale -a` 为您的首选语言环境（在我们的示例中为 `en_GB.ISO88591`）给出的输出。

```sh
# 不要执行这条语句：
LC_ALL=<locale name> locale charmap
# 执行这条语句，将语言设置为中文：
(lfs chroot) root:/# LC_ALL=zh_CN.utf8 locale charmap
UTF-8
```

在将使用上述启发式找到的区域设置添加到 Bash 启动文件之前，必须对其进行测试，这一点很重要：

```sh
(lfs chroot) root:/# LC_ALL=zh_CN.utf8 locale language
Chinese
(lfs chroot) root:/# LC_ALL=zh_CN.utf8 locale charmap
UTF-8
(lfs chroot) root:/# LC_ALL=zh_CN.utf8 locale int_curr_symbol
CNY 
(lfs chroot) root:/# LC_ALL=zh_CN.utf8 locale int_prefix
86
```

上述命令应打印语言名称、语言环境使用的字符编码、当地货币以及在电话号码前拨打的前缀，以便进入该国家/地区。如果上面的任何命令失败并显示类似于下面显示的消息，这意味着您的语言环境未在第 8.8 节“Glibc-2.32”中安装，或者 Glibc 的默认安装不支持。

```sh
locale: Cannot set LC_* to default locale: No such file or directory
```

如果发生这种情况，您应该使用 `localedef` 命令安装所需的语言环境，或者考虑选择不同的语言环境。进一步的说明假定 Glibc 没有此类错误消息。

如果语言环境名称不符合他们的期望，其他包也可能无法正常运行（但不一定会显示任何错误消息）。在这些情况下，调查其他 Linux 发行版如何支持您的语言环境可能会提供一些有用的信息。

确定正确的区域设置后，创建 `/etc/profile`文件：

```sh
# 执行以下操作：
cat > /etc/profile << "EOF"
# Begin /etc/profile

export LANG=<ll>_<CC>.<charmap><@modifiers>

# End /etc/profile
EOF
```

`C`（默认）和 `en_US.utf8` （推荐美国英语用户使用）的语言环境是不同的。 `C` 使用 `US-ASCII` 7位字符集，并将设置高位的字节视为无效字符。这就是为什么 ls 命令会在该语言环境中用问号替换它们。此外，尝试从 Mutt 或 Pine 发送带有此类字符的邮件会导致发送不符合 RFC 的消息（外发邮件中的字符集表示为 `unknown 8-bit`）。因此， 只有当您确定永远不需要 8 位字符时，才可以使用 `C` 语言环境。

### 9.8 创建 /etc/inputrc 文件

inputrc 文件是 readline 库的配置文件，它在用户从终端输入一行时提供编辑功能。它的工作原理是将键盘输入转换为特定的操作。Readline 被 bash 和大多数其他 shell 以及许多其他应用程序使用。

大多数人不需要特定于用户的功能，因此下面的命令会创建一个 `/etc/inputrc` 供所有登录者使用的全局变量。如果您稍后决定需要基于每个用户覆盖默认值，您可以在用户的主目录中创建一个 `.inputrc` 文件修改后的映射。

有关如何编辑 inputrc 文件的更多信息，可以查看 info bash 中 Readline Init File 章节。info readline 也是一个很好的信息来源。

下面是一个通用的全局 inputrc 变量以及解释各种选项的作用的注释。请注意，注释不能与命令在同一行。使用以下命令创建文件：

```sh
# 执行以下命令：
cat > /etc/inputrc << "EOF"
# Begin /etc/inputrc
# Modified by Chris Lynn <roryo@roryo.dynup.net>

# Allow the command prompt to wrap to the next line
set horizontal-scroll-mode Off

# Enable 8bit input
set meta-flag On
set input-meta On

# Turns off 8th bit stripping
set convert-meta Off

# Keep the 8th bit for display
set output-meta On

# none, visible or audible
set bell-style none

# All of the following map the escape sequence of the value
# contained in the 1st argument to the readline specific functions
"\eOd": backward-word
"\eOc": forward-word

# for linux console
"\e[1~": beginning-of-line
"\e[4~": end-of-line
"\e[5~": beginning-of-history
"\e[6~": end-of-history
"\e[3~": delete-char
"\e[2~": quoted-insert

# for xterm
"\eOH": beginning-of-line
"\eOF": end-of-line

# for Konsole
"\e[H": beginning-of-line
"\e[F": end-of-line

# End /etc/inputrc
EOF
```

### 9.9 创建 /etc/shells 文件

shells 文件包含系统上的登录 shell 列表。应用程序使用此文件来确定 shell 是否有效。对于每个 shell，应该有一行，由 shell 相对于目录结构根目录 (/) 的路径组成。

例如， chsh 会查阅此文件以确定非特权用户是否可以更改其自己帐户的登录 shell。如果未列出命令名称，用户将被拒绝更改 shell 的能力。

这是 GDM 等应用程序的要求，如果找不到 `/etc/shells`，则不会填充面部浏览器，或者传统上不允许使用未包含在此文件中的 shell 的用户访问的 FTP 守护程序。

```sh
# 执行以下命令：
cat > /etc/shells << "EOF"
# Begin /etc/shells

/bin/sh
/bin/bash

# End /etc/shells
EOF
```

## 第十章 LFS 系统引导

我在此处创建了一个快照！

### 10.1 概述

是时候让 LFS 系统可启动了。本章讨论创建 `/etc/fstab` 文件、为新的 LFS 系统构建内核以及安装 `GRUB` 引导加载程序，以便在启动时选择 LFS 系统进行引导。

### 10.2 创建 /etc/fstab 文件

某些程序使用 `/etc/fstab` 文件来确定默认情况下将文件系统安装到何处、以何种顺序以及在安装前必须检查哪些文件系统（完整性错误）。创建一个新的文件系统表，如下所示：

```sh
# 不要执行以下命令：
cat > /etc/fstab << "EOF"
# Begin /etc/fstab

# file system  mount-point  type     options             dump  fsck
#                                                              order

/dev/<xxx>     /            <fff>    defaults            1     1
/dev/<yyy>     swap         swap     pri=1               0     0
proc           /proc        proc     nosuid,noexec,nodev 0     0
sysfs          /sys         sysfs    nosuid,noexec,nodev 0     0
devpts         /dev/pts     devpts   gid=5,mode=620      0     0
tmpfs          /run         tmpfs    defaults            0     0
devtmpfs       /dev         devtmpfs mode=0755,nosuid    0     0

# End /etc/fstab
EOF
```

替换 `<xxx>`， `<yyy>` 和 `<fff>` 的值以便适合于系统，例如替换为 `sda2`，`sda5`，和 `ext4`。有关此文件中六个字段的详细信息，请参阅 `man 5 fstab`。

根据我的实际情况，我只挂载了 `/` 分区，因此将命令修改为：

```sh
cat > /etc/fstab << "EOF"
# Begin /etc/fstab

# file system  mount-point  type     options             dump  fsck
#                                                              order

/dev/sdb1      /            ext4     defaults            1     1
proc           /proc        proc     nosuid,noexec,nodev 0     0
sysfs          /sys         sysfs    nosuid,noexec,nodev 0     0
devpts         /dev/pts     devpts   gid=5,mode=620      0     0
tmpfs          /run         tmpfs    defaults            0     0
devtmpfs       /dev         devtmpfs mode=0755,nosuid    0     0

# End /etc/fstab
EOF
```

关于以上分区，可以通过输入以下命令查看：

```sh
(lfs chroot) root:/# df -h
Filesystem      Size  Used Avail Use% Mounted on
/dev/sdb1        20G  1.6G   17G   9% /
udev            3.9G     0  3.9G   0% /dev
tmpfs           3.9G     0  3.9G   0% /run
```

这里的 `udev` 和 `tmpfs` 是在构建 LFS 时生成的分区，除去这两个分区外，就是我们需要配置的分区。

MS-DOS 或 Windows 的文件系统（即 vfat、ntfs、smbfs、cifs、iso9660、udf）需要一个特殊选项 utf8，以便正确解释文件名中的非 ASCII 字符。

对于非 UTF-8 语言环境， iocharset 的值应设置为与语言环境的字符集相同，并以内核能够理解的方式进行调整。如果相关字符集定义（在配置内核时在文件系统 -> 本地语言支持下找到）已编译到内核或构建为模块，则此方法有效。但是，如果语言环境的字符集是 UTF-8，则对应的选项 iocharset=utf8 将使文件系统区分大小写。要解决此utf8问题，需要用特殊选项 utf8 代替 iocharset=utf8。另外，vfat 和 smbfs 文件系统还需要“codepage”选项，它应该被设定为您的语言在 MS-DOS 下的代码页编号。

例如，为了挂载 USB 闪存驱动器，`zh-CN.GBK` 用户需要在 `/etc/fstab` 中添加以下的挂载选项：

```sh
# 我不运行此命令
noauto,user,quiet,showexec,iocharset=gbk,codepage=936
```

对于 `zh_CN.UTF-8` 用户的对应选项是：

```sh
# 我不运行此命令
noauto,user,quiet,showexec,codepage=866,utf8
```

另外，一些硬盘类型在遇到电源故障时，假如在 `/etc/fstab` 中使用 `barrier=1` 这个挂载选项，则会让 ext3 文件系统的数据更加安全。如需检查磁盘是否支持此选项，请运行 hdparm。例如：

```sh
hdparm -I /dev/sda | grep NCQ
```

如果有输出内容，则代表选项可用。

我执行以上命令后的结果：

```sh
(lfs chroot) root:/# hdparm -I /dev/sda | grep NCQ
bash: hdparm: command not found
```

因此忽略该项。

### 10.3 Linux-5.8.3

重要：请注意，本章节耗时非常非常非常久！

Linux 软件包包含 Linux 内核。

#### 10.3.1 安装内核

解压软件包：

```sh
(lfs chroot) root:/# cd /sources/
(lfs chroot) root:/sources# tar xf linux-5.8.3.tar.xz 
(lfs chroot) root:/sources# cd linux-5.8.3
```

构建内核涉及几个步骤 —— **配置**、**编译**和**安装**。阅读README内核源代码树中的文件，了解本书配置内核方式的替代方法。

通过运行以下命令准备编译：

```sh
(lfs chroot) root:/sources/linux-5.8.3# make mrproper
```

这确保内核树是绝对干净的。内核团队建议在每次内核编译之前发出此命令。不要依赖解压后的源代码树是干净的。

有多种方法可以配置内核选项。通常，这是通过菜单驱动的界面完成的，例如：

```sh
# 先不要运行此命令
make menuconfig
```

注意：设置内核配置的一个好的起点是运行 `make defconfig`。这会将基本配置设置为考虑您当前系统架构的良好状态。

请务必启用/禁用/设置以下功能，否则系统可能无法正常工作或根本无法启动：

```
设备驱动程序 --->
  通用驱动程序选项 ---> 
   [ ] 支持 uevent 助手 [CONFIG_UEVENT_HELPER] 
   [*] 维护一个 devtmpfs 文件系统以挂载在 /dev [CONFIG_DEVTMPFS]
```

如果您的主机硬件使用 UEFI，那么上面的 `make defconfig` 应该会自动添加一些与 EFI 相关的内核选项。

为了允许从主机的 UEFI 引导环境中引导 LFS 内核，您的内核必须选择此选项：

```
处理器类型和特性 ---> 
   [*] EFI 存根支持 [CONFIG_EFI_STUB]
```

```sh
# 运行以下命令，并按照样例配置：
(lfs chroot) root:/sources/linux-5.8.3# make defconfig
  HOSTCC  scripts/basic/fixdep
  HOSTCC  scripts/kconfig/conf.o
  HOSTCC  scripts/kconfig/confdata.o
  HOSTCC  scripts/kconfig/expr.o
  LEX     scripts/kconfig/lexer.lex.c
  YACC    scripts/kconfig/parser.tab.[ch]
  HOSTCC  scripts/kconfig/lexer.lex.o
  HOSTCC  scripts/kconfig/parser.tab.o
  HOSTCC  scripts/kconfig/preprocess.o
  HOSTCC  scripts/kconfig/symbol.o
  HOSTCC  scripts/kconfig/util.o
  HOSTLD  scripts/kconfig/conf
*** Default configuration is based on 'x86_64_defconfig'
#
# configuration written to .config
#
```

接下来，可以用 `make menuconfig` 进行配置，并保证按下面的样例进行设置：

```sh
(lfs chroot) root:/sources/linux-5.8.3# make menuconfig

# 请务必保证以下配置项按样例配置：
General setup -->
   [ ] Auditing Support [CONFIG_AUDIT] # 此项需要修改
   [*] Control Group support [CONFIG_CGROUPS]
   [ ] Enable deprecated sysfs features to support old userspace tools [CONFIG_SYSFS_DEPRECATED]
   [*] Configure standard kernel features (expert users) [CONFIG_EXPERT] 
      --->   # 上面这一项需要修改
      [*] open by fhandle syscalls [CONFIG_FHANDLE]
Processor type and features  --->
   [*] Enable seccomp to safely compute untrusted bytecode [CONFIG_SECCOMP]
Firmware Drivers  --->
   [*] Export DMI identification via sysfs to userspace [CONFIG_DMIID]
Networking support  --->
  Networking options  --->
   <*> The IPv6 protocol [CONFIG_IPV6]
Device Drivers  --->
  Generic Driver Options  --->
   [ ] Support for uevent helper [CONFIG_UEVENT_HELPER]
   [*] Maintain a devtmpfs filesystem to mount at /dev [CONFIG_DEVTMPFS]
   Firmware Loader --->
      [ ] Enable the firmware sysfs fallback mechanism [CONFIG_FW_LOADER_USER_HELPER]
File systems  --->
   [*] Inotify support for userspace [CONFIG_INOTIFY_USER]
  Pseudo filesystems  --->
   [*] Tmpfs POSIX Access Control Lists [CONFIG_TMPFS_POSIX_ACL]

# 配置完成后保存退出，会输出以下信息：
*** End of the configuration.
*** Execute 'make' to start the build or try 'make help'.
```

编译内核镜像和模块：

```sh
(lfs chroot) root:/sources/linux-5.8.3# time { make; }

# 编译完成后输出信息：
Setup is 13884 bytes (padded to 14336 bytes).
System is 7542 kB
CRC 5a787df3
Kernel: arch/x86/boot/bzImage is ready  (#1)
  MODPOST Module.symvers
  CC [M]  drivers/thermal/intel/x86_pkg_temp_thermal.mod.o
  LD [M]  drivers/thermal/intel/x86_pkg_temp_thermal.ko
  CC [M]  fs/efivarfs/efivarfs.mod.o
  LD [M]  fs/efivarfs/efivarfs.ko
  CC [M]  net/ipv4/netfilter/iptable_nat.mod.o
  LD [M]  net/ipv4/netfilter/iptable_nat.ko
  CC [M]  net/ipv4/netfilter/nf_log_arp.mod.o
  LD [M]  net/ipv4/netfilter/nf_log_arp.ko
  CC [M]  net/ipv4/netfilter/nf_log_ipv4.mod.o
  LD [M]  net/ipv4/netfilter/nf_log_ipv4.ko
  CC [M]  net/ipv6/netfilter/nf_log_ipv6.mod.o
  LD [M]  net/ipv6/netfilter/nf_log_ipv6.ko
  CC [M]  net/netfilter/nf_log_common.mod.o
  LD [M]  net/netfilter/nf_log_common.ko
  CC [M]  net/netfilter/xt_LOG.mod.o
  LD [M]  net/netfilter/xt_LOG.ko
  CC [M]  net/netfilter/xt_MASQUERADE.mod.o
  LD [M]  net/netfilter/xt_MASQUERADE.ko
  CC [M]  net/netfilter/xt_addrtype.mod.o
  LD [M]  net/netfilter/xt_addrtype.ko
  CC [M]  net/netfilter/xt_mark.mod.o
  LD [M]  net/netfilter/xt_mark.ko
  CC [M]  net/netfilter/xt_nat.mod.o
  LD [M]  net/netfilter/xt_nat.ko

real    34m2.328s
user    30m12.445s
sys     3m10.346s
```

如果使用内核模块，`/etc/modprobe.d` 可能需要模块配置。除非在内核配置中禁用了模块支持，否则安装模块：

```sh
(lfs chroot) root:/sources/linux-5.8.3# make modules_install
  INSTALL drivers/thermal/intel/x86_pkg_temp_thermal.ko
  INSTALL fs/efivarfs/efivarfs.ko
  INSTALL net/ipv4/netfilter/iptable_nat.ko
  INSTALL net/ipv4/netfilter/nf_log_arp.ko
  INSTALL net/ipv4/netfilter/nf_log_ipv4.ko
  INSTALL net/ipv6/netfilter/nf_log_ipv6.ko
  INSTALL net/netfilter/nf_log_common.ko
  INSTALL net/netfilter/xt_LOG.ko
  INSTALL net/netfilter/xt_MASQUERADE.ko
  INSTALL net/netfilter/xt_addrtype.ko
  INSTALL net/netfilter/xt_mark.ko
  INSTALL net/netfilter/xt_nat.ko
  DEPMOD  5.8.3
```

内核映像的路径可能因所使用的平台而异。下面的文件名可以根据您的喜好进行更改，但文件名的主干应该是 vmlinuz 以与下一节中描述的引导过程的自动设置兼容。以下命令假定为 x86 架构：

```sh
(lfs chroot) root:/sources/linux-5.8.3# cp -iv arch/x86/boot/bzImage /boot/vmlinuz-5.8.3-lfs-10.0
'arch/x86/boot/bzImage' -> '/boot/vmlinuz-5.8.3-lfs-10.0'
```

System.map是内核的符号文件。它映射内核 API 中每个函数的函数入口点，以及正在运行的内核的内核数据结构的地址。它在调查内核问题时用作资源。执行以下命令以安装映射文件：

```sh
(lfs chroot) root:/sources/linux-5.8.3# cp -iv System.map /boot/System.map-5.8.3
'System.map' -> '/boot/System.map-5.8.3'
```

内核配置文件 `.config` 由上述的 `make menuconfig` 步骤生成，包含编译好的内核的所有配置选项。最好能将它保留下来以供日后参考：

```sh
(lfs chroot) root:/sources/linux-5.8.3# cp -iv .config /boot/config-5.8.3
'.config' -> '/boot/config-5.8.3'
```

安装 Linux 内核的文档：

```sh
(lfs chroot) root:/sources/linux-5.8.3# install -d /usr/share/doc/linux-5.8.3
(lfs chroot) root:/sources/linux-5.8.3# cp -r Documentation/* /usr/share/doc/linux-5.8.3 
```

#### 10.3.2 配置 Linux 内核模块加载顺序

多数情况下 Linux 内核模块可以自动加载，但有时需要指定加载顺序。负责加载内核模块的程序 `modprobe` 和 `insmod` 从 `/etc/modprobe.d` 下的配置文件中读取加载顺序，例如，如果 USB 驱动程序 (ehci_hcd、ohci_hcd 和 uhci_hcd) 被构建为模块，则必须按照先加载 `echi_hcd`，再加载 `ohci_hcd` 和 `uhci_hcd` 的正确顺序，才能避免引导时出现警告信息。

为此，执行以下命令创建文件 `/etc/modprobe.d/usb.conf`：

```sh
(lfs chroot) root:/sources/linux-5.8.3# install -v -m755 -d /etc/modprobe.d
install: creating directory '/etc/modprobe.d'

# 继续执行以下命令
cat > /etc/modprobe.d/usb.conf << "EOF"
# Begin /etc/modprobe.d/usb.conf

install ohci_hcd /sbin/modprobe ehci_hcd ; /sbin/modprobe -i ohci_hcd ; true
install uhci_hcd /sbin/modprobe ehci_hcd ; /sbin/modprobe -i uhci_hcd ; true

# End /etc/modprobe.d/usb.conf
EOF
```

### 10.4 使用 GRUB 设置引导过程

我在此处创建了快照！

#### 10.4.3 设置配置

将 GRUB 文件安装到 `/boot/grub` 并设置引导轨道：

```sh
(lfs chroot) root:/sources/linux-5.8.3# grub-install /dev/sdb
Installing for i386-pc platform.
Installation finished. No error reported.
```

#### 10.4.4 创建 GRUB 配置文件

生成 `/boot/grub/grub.cfg`：

```sh
# 注意，注意修改以下命令中的 (hd1,1) 和 /dev/sdb1
# 执行以下命令：
cat > /boot/grub/grub.cfg << "EOF"
# Begin /boot/grub/grub.cfg
set default=0
set timeout=5

insmod ext2
set root=(hd1,1)

menuentry "GNU/Linux, Linux 5.8.3-lfs-10.0 (build by ZHJ0125)" {
        linux   /boot/vmlinuz-5.8.3-lfs-10.0 root=/dev/sdb1 ro
}
EOF
```

## 第 11 章结束

### 11.1 尾声

创建/etc/lfs-release文件可能是个好主意。有了这个文件，您（如果您在某个时候需要寻求帮助，我们也很容易）找出系统上安装了哪个 LFS 版本。通过运行创建此文件：

```sh
echo 10.0 > /etc/lfs-release
```

两个描述已安装系统的文件可能会被稍后安装在系统上的软件包使用，无论是二进制形式还是通过构建它们。

第一个显示新系统相对于 Linux 标准库 (LSB) 的状态。要创建此文件，请运行：

```sh
# 执行该脚本
cat > /etc/lsb-release << "EOF"
DISTRIB_ID="Linux From Scratch"
DISTRIB_RELEASE="10.0"
DISTRIB_CODENAME="ZHJ0125"
DISTRIB_DESCRIPTION="Linux From Scratch"
EOF
```

第二个包含大致相同的信息，被 systemd 和一些图形桌面环境使用。要创建此文件，请运行：

```sh
# 执行该脚本
cat > /etc/os-release << "EOF"
NAME="Linux From Scratch"
VERSION="10.0"
ID=lfs
PRETTY_NAME="Linux From Scratch 10.0"
VERSION_CODENAME="ZHJ0125"
EOF
```

请务必对 `DISTRIB_CODENAME` 和 `VERSION_CODENAME` 字段进行某种自定义，以使系统独一无二。

### 11.2 重新启动

既然我们已经说过了，让我们继续第一次启动我们闪亮的新 LFS 安装！首先退出chroot环境：

```sh
(lfs chroot) root:/sources/linux-5.8.3# logout
root@ubuntu:~# 
```

然后卸载虚拟文件系统：

```sh
root@ubuntu:~# umount -v $LFS/dev/pts
umount: /mnt/lfs/dev/pts unmounted
root@ubuntu:~# umount -v $LFS/dev
umount: /mnt/lfs/dev unmounted
root@ubuntu:~# umount -v $LFS/run
umount: /mnt/lfs/run unmounted
root@ubuntu:~# umount -v $LFS/proc
umount: /mnt/lfs/proc unmounted
root@ubuntu:~# umount -v $LFS/sys
umount: /mnt/lfs/sys unmounted
```

如果创建了多个分区，请在卸载主分区之前卸载其他分区，如下所示：

```sh
root@ubuntu:~# umount -v $LFS/usr
umount: /mnt/lfs/usr: not mounted.
root@ubuntu:~# umount -v $LFS/home
umount: /mnt/lfs/home: not mounted.
root@ubuntu:~# umount -v $LFS
umount: /mnt/lfs: target is busy.
root@ubuntu:~# umount -v $LFS
umount: /mnt/lfs: target is busy.
```

卸载 LFS 文件系统本身：

```sh
# 如果出现卸载磁盘失败的情况，如下面所示：
root@ubuntu:~# umount -v $LFS
umount: /mnt/lfs: target is busy.
# 此时可以使用 lsof /mnt/lfs 查看磁盘被占用的情况
# 或者尽量关闭所有可能占用磁盘的应用，如终端应用，关闭之后再运行以下命令：
zhj@ubuntu:~$ sudo umount -v /mnt/lfs/
umount: /mnt/lfs/ unmounted
```

假设 GRUB 引导加载程序按照前面所述进行设置，菜单设置为自动引导LFS 10.0。

重新启动完成后，LFS 系统就可以使用了，并且可能会添加更多软件以满足您的需求。

现在，使用以下命令重新启动系统：

```sh
shutdown -r now
```

终于！

LFS系统构建结束了！

End
