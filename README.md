# LFS

Linux From Scratch (LFS) 构建过程记录

版本: LFS_v10.0

Issue:
* /dev/sda 和 /dev/sdb 使用了同一套 grub.cfg 引导文件，这会导致宿主机无法正常引导。
* 目标机启动后，显示 `No working init found.Try Passing init= option to kernel. See Linux Docurentation/adrin-guide/init.rsc`.
