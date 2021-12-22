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

### 8.10 安装 Bzip2-1.0.8

Bzip2 包包含用于压缩和解压缩文件的程序。使用 bzip2 压缩文本文件比使用传统 gzip 产生更好的压缩百分比。








































