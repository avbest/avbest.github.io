---
title: gpu_passthrough.md
published: 2026-07-09
description: ''
image: ''
tags: []
category: 'Linux'
draft: false 
lang: ''
---

## 注：本教程仅适用至少两个gpu的电脑，如果你只有一个gpu，请**不要**尝试本教程的方法，有可能导致无法开机！

最近终于是把系统换成了cachyos...受不了wsl糟糕的网络问题了，一狠心就给换了. 只能说有点曲折，笔记本装linux很看脸——也和品牌有关，有的牌子笔记本天生就是linux友好的，换了之后驱动也不用操心，即装即用. 但是国内牌子一般就不行，一般都是对linux适配多少存在点问题，比如我这个荣耀的笔记本，装上之后指纹就用不了（我这个是FPC的什么什么，在libfprint里面直接列到不支持列表了，后来找到一个俄罗斯高手做的第三方驱动才搞上，不过还是有很多问题，所以我就放弃指纹了）.

还有音频，这个到现在我也没搞好，这个alc256的螃蟹声卡到linux上工作一直不正常，声音一直发闷，不管怎么调都没用，试了各种预设还是不行. 你说它缺驱动吧，它也不缺，pipewire也支持，声音什么都正常放，唯独就是效果不好. 我预感应该是缺什么预设了，但是我把官网的音频驱动解包之后也只找到几个给耳机用的杜比预设（耳机是正常工作的，音质也正常，只有外放不正常），就是没有给外放的...真奇怪，所以也作罢了，只能搞个easyeffects挂个预设缓解缓解了.

好了，说回正题吧.  搞显卡直通其实对我来说没什么用，因为我没有刚需windows才能打的游戏，所有的游戏需求拜托wine就好了，这次搞显卡直通完全是一种挑战吧我感觉，中途出现各种错误也费了我不少时间，因此记录下来，以示后人.  (((o(*ﾟ▽ﾟ*)o)))

## 开始

首先要明白gpu直通的原理，gpu直通是通过vfio实现的. 这个vfio我就不详细解释了，总之根据文档所说，它就是因为虚拟机有直通硬件的需求才诞生的，我们的场景也正好合适. 

> Why do we want that? Virtual machines often make use of direct device access (“device assignment”) when configured for the highest possible I/O performance. From a device and host perspective, this simply turns the VM into a userspace driver, with the benefits of significantly reduced latency, higher bandwidth, and direct use of bare-metal device drivers

(摘自the linux kernel docs)

这也告诉我们，即使你这个卡在linux上打不上驱动，也是没问题的，因为我们会把这个卡直接通到虚拟机里面，只要虚拟机里面能正常打驱动，显卡就是能用的.

那目标也很简单——把gpu的驱动全部屏蔽，接上vfio的驱动，然后让虚拟机使用即可.

那我们开始吧！

首先确保你已经安装了qemu/kvm相关的全套东西，在arch里面这很简单，qemu-full包含了所有qemu相关的东西. 还有管理虚拟机用到的virt-manager，我们后续会主要和它打交道（你也不想手动写qemu启动脚本吧）

```fish
sudo pacman -S qemu-full virt-manager
```

至于kvm——我相信所有看这篇文章的人的电脑都会支持它的.

不过先等等，虚拟机要想支持好这块显卡，加入一个适配的vbios是很重要的，我们接下来就要用nvflash提取一下vbios啦~

在提取vbios之前，**确保**你的安全启动是关掉的，不然会触发访问限制.

其次，提取之前**不能**有任何驱动在显卡上运行. 也就是这块卡不能被系统使用才行. 要做到这一点，我们编辑一下启动参数，把显卡驱动拉黑即可.

先看看我们的显卡都支持什么驱动吧，执行
```fish
lspci -vv
```

找到你的显卡信息，我这里如下

```
01:00.0 VGA compatible controller: NVIDIA Corporation AD107M [GeForce RTX 4060 Max-Q / Mobile] (rev a1) (prog-if 00 [VGA controller])
        Subsystem: Device 1ee7:204d
        Physical Slot: 11
        Control: I/O+ Mem+ BusMaster+ SpecCycle- MemWINV- VGASnoop- ParErr- Stepping- SERR- FastB2B- DisINTx+
        Status: Cap+ 66MHz- UDF- FastB2B- ParErr- DEVSEL=fast >TAbort- <TAbort- <MAbort- >SERR- <PERR- INTx-
        Latency: 0
        Interrupts: pin B disabled, MSI(X) routed to IRQ 174
        IOMMU group: 17
        Region 0: Memory at 86000000 (32-bit, non-prefetchable) [size=16M]
        Region 1: Memory at 4000000000 (64-bit, prefetchable) [size=8G]
        Region 3: Memory at 4200000000 (64-bit, prefetchable) [size=32M]
        Region 5: I/O ports at 3000 [size=128]
        Capabilities: <access denied>
        Kernel driver in use: nvidia
        Kernel modules: nouveau, nvidia_drm, nvidia

```

可以看到，它支持nouveau, nvidia_drm和nvidia三个驱动模块，我们需要把它们都拉黑.

我用的是limine启动管理器，如果用grub的话步骤会差一点，但基本原理一样. 编辑limine的配置文件

```fish
sudo nvim /etc/default/limine
```

在KERNEL_CMDLINE[default]这一条后面加上
```
module_blacklist=nouveau,nvidia_drm,nvidia
```

内核在启动时遇到这个参数就会拒绝加载所有的驱动模块了.

**不要忘记更新**

```fish
sudo limine-update
```

重启一下，之后再进系统看lspci，你会发现它不会再加载驱动了，显示一个access denied.

之后我们安装nvflash，它在aur里面

```fish
paru -S nvflash
```

使用起来很简单，像这样

```fish
sudo nvflash --save vbios.rom
```

很快vbios就dump下来啦.

好，接下来我们要正式开工啦！

要把显卡直接通到vfio上很简单，不过需要先知道显卡的设备id，而且还要看它所在的pci组上有没有其他的设备.

这里借用一下arch wiki上的脚本~

```bash
#!/bin/bash
shopt -s nullglob
for g in $(find /sys/kernel/iommu_groups/* -maxdepth 0 -type d | sort -V); do
    echo "IOMMU Group ${g##*/}:"
    for d in $g/devices/*; do
        echo -e "\t$(lspci -nns ${d##*/})"
    done;
done;
```

把这个脚本保存下来，执行，你会看到系统中所有pci设备所在的分组和id.

它会输出很多...不过你唯一要看的就是，你的显卡情况

我这里如下

```
IOMMU Group 17 01:00.0 VGA compatible controller [0300]: NVIDIA Corporation AD107M [GeForce RTX 4060 Max-Q / Mobile] [10de:28a0] (rev a1)
```

显然，我这里n卡在第17组，设备id是10de:28a0，并且没有别的设备跟它一个组. 记下这个id.

之后编辑启动参数（和之前一样），在参数后面加上

```
intel_iommu=on vfio_pci.ids=<设备id>
```

**注意**：如果你的显卡还与别的设备在一组，不要忘了把那个设备也通给vfio！设备id之间用逗号隔开即可.

**更新参数之后**重启，再看看lspci，你就会发现你的n卡通到vfio上啦.

之后我们就可以准备虚拟机啦！

## 虚拟机配置

这一步就很简单啦，只需要跟着默认向导，一路走下来...等等，还不行哦.

因为virtmanager创建虚拟磁盘的时候默认就是sata磁盘，这个性能不好. 我们可以这样，先不让它创建磁盘.

![](https://i.imgs.ovh/2026/07/09/6aecdf68dcb9d522079964f467b5c30d.png)

但是先别急哦，这一步如果不加存储的话，它默认会用bios启动，这很不好啦，我们要换成uefi.

在最后一步把这个勾上.

![](https://i.imgs.ovh/2026/07/09/a00e811811a7a880d4be94695cbdb2ea.png)

之后完成，它会弹出界面让我们自由配置，这很好.

首先把固件改成uefi.

然后我们提前准备好virtio的客户机镜像（肯定是win啦），直接上网搜一下virtio然后下一个win的就可以啦.

不要忘记新建一个光驱然后挂上virtio镜像哦~这个很简单的吧，我就不介绍啦.

接下来我们自己添加一个virtio的磁盘，这个性能好一些. 

![](https://i.imgs.ovh/2026/07/09/935f0e54d8470b4b3a1eb0acd0ab067d.png)

然后开始安装就行啦~

一路下一步下一步...到选择磁盘这里.

![](https://i.imgs.ovh/2026/07/09/ddd486c03436ac91a5bb440088979bec.png)

windows是不自带virtio驱动的，所以它一个盘都认不出来，我们需要手动加载一下virtio的驱动.

点击load driver，这个时候会弹出一个消息框，点ok就行，它就会自动扫描我们virtio镜像里面的驱动啦.

我用的是win10，所以选w10的就行.

![](https://i.imgs.ovh/2026/07/09/786ec4b61a138c23dc4303a1dbc620bd.png)

之后就是正常安装，正常进系统~

进了系统之后，不要忘了把virtio里面所有的驱动都装进来哦！只需要打开镜像运行里面的msi就行啦~

好，现在把虚拟机关机，然后我们做后续的配置.

## 配置修改

这一部分参考了两部分内容，分别是 https://github.com/clayfreeman/gpu-passthrough 和 https://github.com/tianocore/edk2/discussions/4662#discussioncomment-6541549 ，在此感谢两位大佬的贡献.

首先我们使用virsh编辑一下配置文件

```fish
sudo EDITOR=nvim virsh edit win10
```

这win10就换成你对应的虚拟机名字啦.

找到cpu部分，把check的值改为partial，之后在里面加上下面这两条

```xml
<feature policy='disable' name='hypervisor'/>
<maxphysaddr mode='passthrough' limit='40'/>
```

第一条是为了禁用hypervisor，避免显卡认出这是虚拟环境. 第二条是解决*DMA mapping failed*报错.

然后在features标签里面，加上下面这部分内容，隐藏kvm.

```xml
<kvm>
  <hidden state='on'/>
</kvm>
```

好，现在我们保存就可以啦.

接下来回到virt-manager，把显卡通进去，像这样

![](https://i.imgs.ovh/2026/07/09/9cb5a3fc13c9736b43809e314e2f6747.png)

通进去之后，先别开机，转到这个设备的xml视图，加上下面这条

```xml
<rom file='（vbios路径）' />
```

好啦，到这一步显卡直通其实就已经完成啦，但是这样配出来其实是没法输出显示画面的，我们还需要搭配looking glass来使用才行~这也是本教程的核心哦

## Looking Glass配置

这一步其实跟着官方文档走也很快能搞完，但是考虑到你都来看教程了，我就顺便一起写了吧~

在一切开始之前，先确保虚拟机处于关机状态哦.

首先我们了解一下它的原理，looking glass通过内存共享来完成画面输出，摆脱了网络的带宽限制，效率其实是比sunshine之类的网络串流方案高得多的，当然这也就意味着它只能用作本机虚拟机画面输出，不过我们需要的正是这个.

在安装之前，我们需要先决定looking glass拿多少内存来共享，这一步我们从官网拿公式过来看

<math xmlns="http://www.w3.org/1998/Math/MathML" display="block">
  <mtable displaystyle="true" columnalign="right" columnspacing="" rowspacing="3pt">
    <mtr>
      <mtd>
        <mtable displaystyle="true" columnspacing="" rowspacing="3pt">
          <mtr>
            <mtd>
              <mtext>WIDTH</mtext>
              <mo>&#xD7;</mo>
              <mtext>HEIGHT</mtext>
              <mo>&#xD7;</mo>
              <mtext>BPP</mtext>
              <mo>&#xD7;</mo>
              <mn>2</mn>
              <mo>=</mo>
              <mtext>frame size in bytes</mtext>
            </mtd>
          </mtr>
          <mtr>
            <mtd>
              <mtext>frame size in bytes</mtext>
              <mo>&#xF7;</mo>
              <mn>1024</mn>
              <mo>&#xF7;</mo>
              <mn>1024</mn>
              <mo>=</mo>
              <mtext>&#xA0;frame size in MiB</mtext>
            </mtd>
          </mtr>
          <mtr>
            <mtd>
              <mtext>frame size in MiB</mtext>
              <mo>+</mo>
              <mn>10</mn>
              <mo>=</mo>
              <mtext>&#xA0;required size in MiB</mtext>
            </mtd>
          </mtr>
          <mtr>
            <mtd>
              <msup>
                <mn>2</mn>
                <mrow data-mjx-texclass="ORD">
                  <mo fence="false" stretchy="false">&#x2308;</mo>
                  <msub>
                    <mi>log</mi>
                    <mn>2</mn>
                  </msub>
                  <mo data-mjx-texclass="NONE">&#x2061;</mo>
                  <mo stretchy="false">(</mo>
                  <mtext>required size in MiB</mtext>
                  <mo stretchy="false">)</mo>
                  <mo fence="false" stretchy="false">&#x2309;</mo>
                </mrow>
              </msup>
              <mo>=</mo>
              <mtext>&#xA0;total MiB</mtext>
            </mtd>
          </mtr>
        </mtable>
      </mtd>
    </mtr>
  </mtable>
</math>

就是说，实际需要的内存是，宽 * 高 * bpp * 2 / (2 ^ 20) + 10，然后把这个数取最近的2的幂.

实际算一下，我的屏幕是3072*1920，bpp根据屏幕的hdr来算，如果支持hdr就取8，不支持就取4.

也就是...算一下

```fish
^^>>> math "3072 * 1920 * 8 * 2 / (2 ^ 20) + 10"                                               13:44:15 
100

```

算出来是100，近似一下就取128吧，记住这个数字.

接下来要配置内核模块来完成内存共享，我们直接采用kvmfr方案了，因为这是给独显直通用的.

首先这个kvmfr在aur里其实已经有了，直接从那里面安装就可以了...不过我当时没发现，所以还是手动安装了，因此这里也手动安装.

首先把looking glass的源码clone下来，这个kvmfr在module文件夹里面，直接安装即可

```fish
git clone --recursive https://github.com/gnif/LookingGlass.git
cd LookingGlass/module
sudo dkms install .
```

安装好之后进行配置，新建/etc/modprobe.d/kvmfr.conf，然后写入以下内容

```
options kvmfr static_size_mb=128
```

我这里是128，你要把它换成你自己的数值.

为了使这个模块开机自启，新建/etc/modules-load.d/kvmfr.conf，写入以下内容

```
# KVMFR Looking Glass module
kvmfr
```

然后加载这个模块，让它运行起来

```fish
sudo modprobe kvmfr
```

如果它成功启动了，你应该能看到它的设备.

![](https://i.imgs.ovh/2026/07/09/5fc44a886fc22c8a9f38861b952f9019.png)

一定要确保权限最开头是c，这样才能继续.

然后我们要允许自己使用这个设备，我们使用udev完成这一点，新建/etc/udev/rules.d/99-kvmfr.rules然后写入以下内容

```
SUBSYSTEM=="kvmfr", OWNER="user", GROUP="kvm", MODE="0660"
```

把user换成你自己的用户名，在此之前确保你已经在kvm组里了.

接着我们继续编辑虚拟机配置文件，这一步可以在virt-manager里完成.

在概况里面转到xml视图，在domain后面添加以下属性

```xml
xmlns:qemu='http://libvirt.org/schemas/domain/qemu/1.0'
```

然后在domain的最底部添加以下内容

```xml
<qemu:commandline>
  <qemu:arg value="-device"/>
  <qemu:arg value="{'driver':'ivshmem-plain','id':'shmem0','memdev':'looking-glass'}"/>
  <qemu:arg value="-object"/>
  <qemu:arg value="{'qom-type':'memory-backend-file','id':'looking-glass','mem-path':'/dev/kvmfr0','size':33554432,'share':true}"/>
</qemu:commandline>
```

注意，这里面的size的值需要根据实际计算，就是先前那个数字乘以2^20，我这里算出来是134217728.

应用即可.

之后新建/etc/apparmor.d/local/abstractions/libvirt-qemu，写入以下内容

```
# Looking Glass
/dev/kvmfr0 rw,
```

之后编辑/etc/libvirt/qemu.conf，找到cgroup_device_acl，取消注释这一整块，把/dev/kvmfr0加进去，像这样

```
cgroup_device_acl = [
    "/dev/null", "/dev/full", "/dev/zero",
    "/dev/random", "/dev/urandom",
    "/dev/ptmx", "/dev/userfaultfd",
    "/dev/kvmfr0",
]
```

然后重启一下libvirt服务

```fish
sudo systemctl restart libvirtd.service
```

之后我们配置一下其他的事项，looking glass是借助spice协议来完成键鼠输入的.

首先确保你已经有一个spice的显示协议（graphics）了，把显卡从默认的qxl改为vga. 之后移除任何tablet类型的设备（在virt里面这个叫绘图板），然后创建两个virtio的设备，一个鼠标一个键盘.

（注：virt里面没有virtio鼠标，但是有virtio键盘. 对于鼠标，我们先选一个usb的，然后在xml视图把usb改成virtio即可.）


确保你已经有了一个ich9的声卡设备，一个spice信道.

然后把memballoon的model值改成none，这个功能会严重影响vfio直通的性能.

好，现在一切就绪，开机！

开机之后我们直接上网搜spice tools，找到windows binaries，下载一个给虚拟机装上

![](https://i.imgs.ovh/2026/07/09/aee44e22db8750ed5303c05e97e3f97e.png)

可能网速会有点慢，如果你等不及的话，可以在主机上下好，然后打成iso通给虚拟机.

装好这个之后，我们先看下设备管理器，看看显卡通没通进去.

![](https://i.imgs.ovh/2026/07/09/fc64da01261124ba8df6788b1b71a2f7.png)

看看这里的设备id，如果和主机对得上就是通进去了.

（不知道为什么我这里直接认出来了，你那边第一次装应该是认不出来的）

接下来打一下显卡驱动就好啦，这一步我就不说啦，很容易.

打好驱动之后，**一定要重启一次**. 重启之后，我们继续搜一下looking glass，在虚拟机里把它的host端装上. 这个我也不提了，一路下一步~

不过呐，这样装好之后还是不能用的~因为n卡还没给任何一个显示器输出画面（虚拟机那个可算不上），所以windows还没有让它工作. 我们需要装一个vdd（virtual display driver），欺骗windows，让它误以为已经有一个显示器分配给n卡了，这样就工作啦.

vdd安装也是十分傻瓜式，github上下载下来，运行主程序，然后点intall driver就行啦~

不过它默认的显示配置可能并不适合你的电脑，如果没有适合你的，就自己改一下.

在C:\VirtualDisplayDriver，里面有一个xml文件，照葫芦画瓢加一个你自己的显示配置就好啦

![](https://i.imgs.ovh/2026/07/09/4d786a1c187f39ace831d606bf8ede8c.png)

改好之后重启一下.

别忘了在主机上装好looking glass的client端哦~arch的源已经包含啦.

```fish
sudo pacman -S looking-glass-rc
```

之后，最关键的一步，在windows设置里面，把自带的那个虚拟机显示设备（wired display）直接禁用掉.

![](https://i.imgs.ovh/2026/07/09/acccadbe516d3abbf96f5e676fbfab97.png)

把这个开关打开就好啦~

到这里你的looking glass应该会正常工作啦~好耶！

## 总结

摸索出这一套方法真是浪费了我好长好长时间~不过搞好了之后，还是很有成就感的！当然最该感谢的还得是辛苦研究这些技术的大佬们，真的很感谢~

哦对了，你可能会问，怎么把直通恢复回来.

这很好办，既然它通了vfio，那我们就把vfio模块卸载掉，然后再加载n卡的驱动模块. 

```fish
sudo rmmod vfio_pci
sudo modprobe nvidia
sudo modprobe nvidia_drm
```
（PS：需要注意，这种手动恢复的方式似乎不能被wine响应，因此最好还是改启动参数再重启比较好哦！）
