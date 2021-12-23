# LFS

Linux From Scratch (LFS) 构建过程记录

版本: LFS_v10.0_SystemV

当前存在的问题:
* /dev/sda 和 /dev/sdb 使用了同一套 `/boot/grub/grub.cfg` 引导文件，这会导致宿主机无法正常引导。
* 目标机启动后，显示 `No working init found.Try Passing init= option to kernel. See Linux Docurentation/adrin-guide/init.rsc`.

解决方案：
* 使用虚拟机快照将系统状态恢复至 GRUB 配置之前(Snapshot 5)
* 继续尝试关于 GRUB 和 SystemV 的配置
