
目录* [安装虚拟机系统](https://github.com)
* [共享目录](https://github.com)
* [编译内核](https://github.com)
* [卸载内核](https://github.com)
* [参考资料](https://github.com)

本文主要记录个人做存储系统研究时，在 QEMU 环境下编译和安装 Linux 内核的过程


## 安装虚拟机系统


之前在 [利用 RocksDB \+ ZenFS 测试 ZNS 的环境搭建和使用](https://github.com) 给出过借助 VNC 进行图形化安装的步骤，这里再给出仅通过终端进行安装的步骤



```


|  | # 下载 Ubuntu 镜像 |
| --- | --- |
|  | wget https://releases.ubuntu.com/24.04.1/ubuntu-24.04.1-live-server-amd64.iso |
|  |  |
|  | # 制作磁盘镜像，大小随意 |
|  | qemu-img create -f qcow2 u24s.qcow2 80G |
|  |  |
|  | # ubuntu 镜像挂在 cdrom 上启动 |
|  | # -enable-kvm 用于开启 KVM 虚拟化 |
|  | # -boot once=d 用于只从 cdrom 启动一次 |
|  | # -nographic 用于关闭图形界面 |
|  | qemu-system-x86_64 -m 8G -smp 4 -enable-kvm -nographic -hda u24s.qcow2 \ |
|  | -cdrom ubuntu-24.04.1-live-server-amd64.iso -boot once=d |


```

![按 e 进入编辑模式](https://i-blog.csdnimg.cn/direct/9289e4a4e5e4431e9c77e63fcf19317d.png#pic_center)


然后在 grub menu 按 `e` 进入编辑模式


![新增 console=ttyS0](https://i-blog.csdnimg.cn/direct/7d1d97aedd9e4fc9809efe43272d6388.png#pic_center)


然后在 vmlinuz 那一行新增 `console=ttyS0`，之后 `ctrl+x` 启动即可


安装完毕后，后续启动命令可以简化



```


|  | qemu-system-x86_64 -m 8G -smp 4 -enable-kvm -nographic -hda u24s.qcow2 |
| --- | --- |


```

但是此时的启动过程中的 grub menu 不会显示，还需要修改下 grub 配置



```


|  | sudo vim /etc/default/grub |
| --- | --- |
|  |  |
|  | # 修改下面三个配置项 |
|  | #GRUB_TIMEOUT_STYLE=hidden |
|  | GRUB_TIMEOUT=3 |
|  | GRUB_TERMINAL=console |
|  |  |
|  | sudo update-grub |
|  | sudo poweroff |


```

如果想通过 ssh 登陆虚拟机，启动参数可以加一个端口转发



```


|  | qemu-system-x86_64 -m 8G -smp 4 -enable-kvm -nographic -hda u24s.qcow2 \ |
| --- | --- |
|  | -net nic,model=virtio -net user,hostfwd=tcp::6666-:22 |


```

之后就可以在物理机器上通过 ssh 登陆虚拟机了



```


|  | ssh -p 6666 [user]@localhost |
| --- | --- |


```

## 共享目录


为了加速内核编译，可以在物理机器上编译内核，然后将编译好的内核文件借助共享目录传输到虚拟机中



```


|  | # 在物理机器上创建共享目录 |
| --- | --- |
|  | mkdir -p xxx/share |
|  |  |
|  | # 启动虚拟机时挂载共享目录 |
|  | qemu-system-x86_64 -m 8G -smp 4 -enable-kvm -nographic -hda u24s.qcow2 \ |
|  | -fsdev local,path=xxx/share,id=share_dir,security_model=none \ |
|  | -device virtio-9p-pci,fsdev=share_dir,mount_tag=hostshare \ |
|  | -net nic,model=virtio -net user,hostfwd=tcp::6666-:22 |


```

如果报错，很有可能是 `qemu` 不支持 `9p`，需要从源码编译 `qemu`，在 `configure` 时加上 `--enable-virtfs` 选项即可


之后在虚拟机中挂载共享目录



```


|  | # 虚拟机中挂载共享目录 |
| --- | --- |
|  | sudo mkdir -p /mnt/share |
|  | sudo mount -t 9p -o trans=virtio hostshare /mnt/share/ -oversion=9p2000.L |


```

如果报错，很有可能是虚拟机的内核不支持 `9p`，需要编译内核，是打开以下内核配置选项：



```


|  | CONFIG_NET_9P=y |
| --- | --- |
|  | CONFIG_NET_9P_VIRTIO=y |
|  | CONFIG_NET_9P_DEBUG=y (Optional) |
|  | CONFIG_9P_FS=y |
|  | CONFIG_9P_FS_POSIX_ACL=y |
|  | CONFIG_PCI=y |
|  | CONFIG_VIRTIO_PCI=y |
|  |  |
|  | CONFIG_PCI=y |
|  | CONFIG_VIRTIO_PCI=y |
|  | CONFIG_PCI_HOST_GENERIC=y (only needed for the QEMU Arm 'virt' board) |


```

## 编译内核


在物理机上准备环境



```


|  | # 编译工具，词法语法分析库 |
| --- | --- |
|  | sudo apt install build-essential bison flex |
|  | # 如果编译时缺少 openssl 的相关头文件，需要安装相关库 |
|  | sudo apt install libssl-dev |
|  | # 利用 make menuconfig 图形界面配置编译选项需要安装 ncurses 环境： |
|  | sudo apt install libncurses5-dev |
|  |  |
|  | # 下载 kernel 源码，解压 |
|  | wget https://cdn.kernel.org/pub/linux/kernel/v5.x/linux-5.4.xxx.tar.xz |
|  | tar xvf linux-5.4.xxx.tar.xz |
|  | mv linux-5.4.xxx xxx/share/ |


```

在虚拟机内获取内核配置



```


|  | sudo mount -t 9p -o trans=virtio hostshare /mnt/share/ -oversion=9p2000.L |
| --- | --- |
|  |  |
|  | cd /mnt/share/linux-5.4.xxx |
|  | sudo make oldconfig |


```

在物理机上编译内核



```


|  | # 解决 make Error 问题 |
| --- | --- |
|  | sudo scripts/config --set-str SYSTEM_TRUSTED_KEYS "" |
|  | sudo scripts/config --set-str SYSTEM_REVOCATION_KEYS "" |
|  |  |
|  | # 编译内核和模块, -j24 表示使用 24 个线程编译, 可以根据自己的 CPU 核心数和内存大小调整 |
|  | sudo make -j24 |


```

在虚拟机内安装内核



```


|  | # 去除调试信息，解决 initrd.img 过大的问题 |
| --- | --- |
|  | sudo make INSTALL_MOD_STRIP=1 modules_install |
|  | sudo make install |
|  | sudo poweroff |


```

## 卸载内核


开发过程中可能会有 bug，需要在虚拟机卸载有问题的内核



```


|  | # 删除 /lib/modules/ 目录下以内核的版本号为名称的目录 |
| --- | --- |
|  | sudo rm -rf /lib/modules/5.4.xxx+/ |
|  |  |
|  | # （可选）删除 /usr/src/linux/ 目录下不需要的内核源码 |
|  | # sudo rm -rf /usr/src/linux-headers-5.4.xxx |
|  |  |
|  | # 删除 /boot 目录下启动的内核和内核映像文件 |
|  | sudo rm /boot/*5.4.xxx* |
|  |  |
|  | # 更改 grub 的配置文件，删除不需要的内核启动列表 |
|  | sudo update-grub2 |


```

## 参考资料


* [【QEMU】Invocation](https://github.com):[樱花宇宙官网](https://yzygzn.com)
* [【QEMU】Invocation](https://github.com)
* [【ask ubuntu】No rule to make target 'debian/canonical\-certs.pem'](https://github.com)
* [【CSDN】qemu 中使用 9p virtio, 支持 host 和 guest 中共享目录](https://github.com)
* [【CSDN】解决 Linux 编译内核模块 (initrd.img) 过大的问题](https://github.com)



> **本文作者：** ywang\_wnlo
> **本文链接：** [https://ywang\-wnlo.github.io/posts/5fce01ae/](https://github.com)
> **版权声明：** 本博客所有文章除特别声明外，均采用 [BY\-NC\-SA](https://github.com) 许可协议。转载请注明


