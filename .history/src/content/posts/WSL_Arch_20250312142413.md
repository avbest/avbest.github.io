---
title: WSL_Arch.md
published: 2025-03-12
description: ''
image: ''
tags: []
category: ''
draft: false 
lang: ''
---
>
> 我是2018年加入windows insider的，算是赶上了wsl1的末班车，因为据wikipedia记载，2019年6月就出wsl2了
>说实话，当时折腾wsl并不是一件简单的事，你需要安装很多可能你从未见过的软件，很多时候也会报一些玄学问题，比如代理什么的，当时想让wsl走主机代理都是一件麻烦的事.
>现在是2025/1/22了，在写这篇文章的时候，wsl已经发展的足够强大了，它的门槛降低了非常多，甚至很多地方你都不需要看教程，自己摸索摸索就出来了.
>怎么说呢，算是我固执的想法吧，我不想让我从前的折腾经历和辛苦整理出的文稿，在“大模型时代”被彻底遗忘.
>

*尚未更完，长期更新ing...*

**本文可能有错误，当你在阅读过程中产生疑问或质疑时，请务必自行查询，并得出正确的答案！**

<h1>目录</h1>
<a href="#F1"><h2>1. 前言</h2></a>
<h2>2. 安装</h2>
<ol>
    <li><a href="#S1">启用WSL</a></li>
    <li><a href="#S2">安装发行版</a></li>
    <li><a href="#S3_O">（可选）移动WSL</a></li>
</ol>
<h2>3. 配置</h2>
<ol>
    <li><a href="#T1">配置源</a></li>
    <li><a href="#T2">终端配置</a></li>
    <li><a href="#T3">vim配置</a></li>
</ol>

<h2>3. 常用工具安装</h2>
<ol>
    <li><a href="#O1">常用小工具</a></li>
    <li><a href="#O2">CUDA</a></li>
</ol>

<a href="#A"><h2>4. FAQ</h2></a>

<h2 id="F1">1. 前言</h2></a>

WSL全称是Windows Subsystem for Linux, 就是在windows下面运行linux. 
在wsl1时，它的实现还是api转译（说实话我还挺佩服这种转译的，它的难度并不低，有点像反向的wine了），把linux上的api转化成winapi来执行，所以很多需要内核层配合的应用它都跑不了.
wsl2则是走了完全不同的路，它采用自家的hyper-v虚拟化技术跑一个完整的linux（其实就是虚拟机），这样一来虽然占用变大了，但它从此获得了完整的内核层支持. 社区的反映表明，这是完全正确的选择. wsl2把虚拟机和主机的集成度做到了新的高度，这是以前从未有过的.
另外，wslg也已经广泛投入使用了，它可以不用额外配置就直接运行wsl里面的gui应用，虽然还有点小问题，但已经是巨大的进步了.
但wsl2有没有缺点呢？当然有. 它的本质是虚拟机，当然也会有虚拟机共有的缺点：和主机的io通信效率低下.
实际上它和主机的io通信效率比vmware一众的虚拟机还要低，尤其是在小文件传输方面，慢到令人发指. 不过人们还是研究出了避免的办法：要么专门开一个虚拟磁盘，让wsl和主机分别读写；要么把小文件打包成大文件，这样就会快很多.
那么这么好的功能，该怎么安装到自己的电脑上呢？

<h2 id="S1">2.1. 启用WSL</h2></a>

首先确保你的电脑支持虚拟化，因为我们只介绍wsl2的安装，wsl1已经不再受支持了.
怎么看支不支持虚拟化呢？其实完全可以默认支持了，但凡新一点的机器都是默认启用虚拟化的.
查看方法就是打开任务管理器，然后在性能页面看cpu的页面，在下面的详细信息里有虚拟化一栏
![ceF0.png](https://i.imgs.ovh/2025/01/22/ceF0.png)

右下角，虚拟化，就可以看到有没有了
如果你的电脑这里确实显示不支持，那还是得去自行查找解决办法了，因为我确实没碰到
紧接着我们打开控制面板，找到程序->程序和功能下面的启用或关闭Windows功能，把最下面的“适用于Linux的Windows子系统”选上，然后确定，等它完成再重启就可以啦（一定要重启！更改windows功能的行为不重启是不生效的！）
![cH2r.png](https://i.imgs.ovh/2025/01/22/cH2r.png)
重启之后打开终端（推荐使用windows terminal，它的使用我想未来会出一篇介绍的），输入wsl --version，你应该能看到版本信息，那就是正常安装了.
值得注意的是，部分模拟器类软件（尤其是安卓模拟器）使用的虚拟化技术与hyper-v冲突，也必然和wsl冲突了. 如果你有这种软件的话，它们是只能留其中一个的，需要注意一下.
<h2 id="S2">2.2. 安装发行版</h2></a>

有了WSL功能还不够，它只是个基础平台，你还得有个具体的发行版才算系统呀
微软商店里面其实已经有很多wsl发行版了，目前来看基本上所有常见的linux发行版都可以从商店安装啦
（刚开始其实就只有ubuntu, kali, debian这种的，后来大家都把自己喜欢的linux发行版打包起来移植到wsl上了，现在已经不愁自己喜欢的发行版在wsl上没包了）
但是有一个比较麻烦的问题还没解决，当然如果你系统盘空间够的话就不用看接下来的这一小节啦
<h2 id="S3_O">（可选）移动WSL</h2>

WSL默认是安装在系统盘的，但我们都知道系统盘属于那种又不想分多又特别容易吃满的盘，所以一般都不会把大件放在系统盘，那怎么挪走WSL呢？
这就要说一个软件了，LxRunOffline
这个就是专门管理WSL的软件，当然也可以挪走WSL，它是一个开源项目，具体链接放在这里
https://github.com/DDoSolitary/LxRunOffline
怎么用呢？为了满足极端耗时主义的好奇心，我决定在文章里面也写上编译的过程
（我知道的，有些人就是喜欢花很多时间在配置环境和编译上，这样就有很多时间有理由摸鱼了，比如我）
先clone下来
```powershell
git clone https://github.com/DDoSolitary/LxRunOffline
```
在这里我们只介绍msvc版本的编译步骤，因为不习惯mingw
需要用到vcpkg
（vcpkg是微软开源的c/c++库管理方案，适用于大多数c/c++生产环境，我会在之后的文章里介绍它）
先更新好vcpkg
```powershell
cd E:\vcpkg
git pull
vcpkg upgrade --no-dry-run
bootstrap-vcpkg
```
然后我们安装必要的库
```powershell
vcpkg install --triplet x64-windows-static libarchive boost-program-options boost-format boost-algorithm boost-test tinyxml2
```
然后打开一个x64 Native Tools窗口（前提是先装了一个任意版本的vs），在里面新建一个build目录，cd进去，然后构建编译
```powershell
mkdir build
cd build
cmake .. -G "NMake Makefiles" -DCMAKE_TOOLCHAIN_FILE=E:\vcpkg\scripts\buildsystems\vcpkg.cmake -DVCPKG_TARGET_TRIPLET=x64-windows-static
nmake
```
**如果你遇到编译报错，请参考以下方案**
如果你遇到了这样的错误
```
E:\GitProj\LxRunOffline\src\lib\include\LxRunOffline\ntdll.h(11): error C2371: “FILE_CASE_SENSITIVE_INFORMATION”: 重定义；不同的基类型
```
则改动对应的头文件，把重复定义的部分删掉.
比如这里，我就需要注释（其实直接删掉就行）掉重定义的部分.
![fUwr.png](https://i.imgs.ovh/2025/01/22/fUwr.png)

然后再编译一次，就可以啦
生成的可执行文件在src/LxRunOffline里面

在迁移之前，我们需要先至少启动一次发行版，这是为了让wsl生成虚拟磁盘文件，好迁移. 安装完成之后执行wsl --shutdown关机
然后我们用LxRunOffline迁移，这里我想把它迁移到D:\Arch里面，我就这么执行
```powershell
.\LxRunOffline.exe m -n Arch -d D:\Arch
```

我这里是一次迁移成功了，如果你有出错的话，建议去repo的issue页面查看，你想问的这里都有.

<h2 id="T1">配置源</h2>

到手肯定先换源啦，我用的是清华源
根据清华源页面的描述https://mirrors.tuna.tsinghua.edu.cn/help/archlinux/
我们编辑/etc/pacman.d/mirrorlist的内容（Arch是预装vim的，不用找别的编辑器啦）
在最上面添加这个
```
Server = https://mirrors.tuna.tsinghua.edu.cn/archlinux/$repo/os/$arch
```

然后sudo pacman -Syyu更新一下
然后添加archlinuxcn源，编辑/etc/pacman.conf，在里面添加
```
[archlinuxcn]
Server = https://mirrors.tuna.tsinghua.edu.cn/archlinuxcn/$arch
```

然后执行sudo pacman -Sy archlinuxcn-keyring签名一下
（PS：据清华源页面描述，这里应该还有一个信任环节，但在我这里并不需要，所以先不写啦）
然后添加arch4edu
执行下面的命令
```sh
sudo pacman-key --recv-keys 7931B6D628C8D3BA
sudo pacman-key --finger 7931B6D628C8D3BA
sudo pacman-key --lsign-key 7931B6D628C8D3BA
```

然后继续向/etc/pacman.conf里面添加
```
[arch4edu]
Server = https://mirrors.tuna.tsinghua.edu.cn/arch4edu/$arch
```

再次更新

如果你需要一些渗透测试工具的话，需要添加blackarch源
向/ec/pacman.conf里面添加
```
[blackarch]
Server = https://mirrors.tuna.tsinghua.edu.cn/blackarch/$repo/os/$arch
```
有一些软件依赖32位的库，我们还需要取消注释multilib部分
值得注意的是，我这个arch在文件里根本没有multilib部分，这就需要参照wiki自己手动添加啦
之后导入blackarch的pgp
```sh
sudo pacman -Sy blackarch-keyring
```
在这里可能会遇到这样的报错
```
error: blackarch: signature from "Levon 'noptrix' Kayan (BlackArch Developer) <noptrix@nullsecurity.net>" is unknown trust
error: failed to synchronize all databases (invalid or corrupted database (PGP signature))
```

然后安装yay, 我们得能用AUR啊
```
sudo pacman -S yay
```
这其实需要本地信任，我们得执行这样的命令
```sh
sudo pacman-key --lsign-key "noptrix@nullsecurity.net"
```
然后再次安装

（如果你的arch的pacman输出没有颜色而你恰好需要的话
编辑/etc/pacman.conf，把第93行Color取消注释
如果你需要进度条的话，把第94行NoProgressBar注释掉）

<h2 id="T2">终端配置</h2>

我们仍然采用zsh.
先把oh-my-zsh拿下来
```sh
sh -c "$(curl -fsSL https://raw.githubusercontent.com/ohmyzsh/ohmyzsh/master/tools/install.sh)"
```

不知道选什么主题好，在网上参照一片博客找的powerlevel10k
这个主题要求使用一个Nerd font, 这不是shell管的，而是渲染终端的应用管的了，你可以自行寻找设置，我使用的是CodeNewRoman Nerd Font.
根据repo页面描述，执行如下命令部署主题
```sh
git clone --depth=1 https://github.com/romkatv/powerlevel10k.git ~/powerlevel10k
echo 'source ~/powerlevel10k/powerlevel10k.zsh-theme' >>~/.zshrc
```

退出后再次打开终端会进入主题设置向导，跟着步骤走就可以啦
然后是常用插件，这里有几款比较常用的，也是从网上博客看的
1. zsh-autosuggestions
> 这个插件提供自动补全功能，应该很多人就是奔着这个来用zsh的吧
> 这里注意要手动安装，从软件源安装的是无法使用的
> ```sh
> git clone https://github.com/zsh-users/zsh-autosuggestions ~/.zsh/zsh-autosuggestions
> source ~/.zsh/zsh-autosuggestions/zsh-autosuggestions.zsh
> ```
2. zsh-syntax-highlighting
> 提供语法高亮，这个也很常用的，它和上面的搭配非常好
> 同样需要手动安装
> 这个插件依赖于插件管理器，因为我们用的是oh-my-zsh，所以执行以下命令
> ```sh
> git clone https://github.com/zsh-users/zsh-syntax-highlighting.git ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-syntax-highlighting
> ```
> 然后编辑~/.zshrc，激活插件
> 找到plugins行，添加zsh-syntax-highlighting

<h2 id="T3">vim配置</h2>

如果放以前的话，我会在这里大篇幅介绍vimplus，还有它的一堆问题解决方案，但现在有lunarvim了，没必要折腾vim了，直接用neovim吧
先做好前置准备
```sh
sudo pacman -S neovim git make python python-pip npm nodejs cargo lazygit --needed
```

然后执行官网的一键安装命令
```sh
LV_BRANCH='release-1.4/neovim-0.9' bash <(curl -s https://raw.githubusercontent.com/LunarVim/LunarVim/release-1.4/neovim-0.9/utils/installer/install.sh)
```

之后提示什么就全默认就可以啦，过一会儿就自动安装完了
记得把.local/bin添加到PATH里哦~
可能安装到最后会有报错，记得第一次运行lvim时执行:Lazy sync

<h2 id='O1'>常用小工具</h2>

### batcat

batcat是一个带有语法高亮和git集成的开源的“cat增强版”，很多时候如果需要一目了然的舒适打印的话，用它就对啦
```sh
sudo pacman -S bat
```
对应的命令是bat

### tldr

tldr-pages是一个社区维护的常用命令用法集合页面，而tldr就是查询这些用法的工具.
这当然很好，现在的人能认真看进去manpage的才几个，真正能落实到使用的又有多少呢
```sh
sudo pacman -S tldr
```
由于它完全靠社区维护，所以稍微冷门一点的工具，它其实都不知道用法的
不过如果你懂得多的话，也欢迎贡献一份力量哦

### chrome

firefox是一个开源的浏览器，是非chromium流的唯一存活流派.
我们本来是想安装这个的，但经过尝试发现firefox在wsl上问题非常多：不支持硬件加速，不能正常调用pulseaudio，点击按钮没反应之类的.
到最后折腾来折腾去还是换成了chrome，这个就改善很多很多了.
```sh
sudo pacman -S google-chrome
```

对应的命令是google-chrome-stable

因为现在已经有wslg了，所以我们不再介绍运行gui应用的步骤，直接运行就可以啦

### htop

htop是一个开源的对标top的工具，原生的top使用起来并不直观，而htop就很好的解决了这一点
```sh
sudo pacman -S htop
```

unzip是info-zip里的一个软件，是一个开源的zip解压缩软件.
```sh
sudo pacman -S unzip
```

### wslpath

插句题外话，wsl里面还有一个很实用的工具，叫wslpath，它是每个wsl发行版都自带的.
具体来讲，wslpath把一个windows路径转化成对应的wsl内linux风格的路径，有了这个，就不用手打啦

只需要记住一个用法，wslpath后面跟windows风格的路径就可以啦

### docker

wsl里面是可以用docker的！只不过docker的话，因为本身就是依托虚拟化的，所以微软和docker官方联合搞了个wsl2集成
在windows上安装好docker，然后在设置里打开wsl2的集成
（实际上如果你本机已经装了wsl的话，docker是根本不会启用它自己的虚拟化方案的，全部用wsl来完成）
不需要在wsl内安装docker，直接在里面运行windows的docker就可以啦，前提是要先把win端的docker启动.

### binwalk

binwalk是一个开源的固件分析工具，通常我们用它来检测一个文件里是否有隐藏的信息.
```sh
sudo pacman -S binwalk
```

<h2 id="O2">CUDA</h2>

CUDA大家应该都不陌生了吧，要用n卡做计算就得用这个.
所幸wsl已经普及了很长时间，现在n卡驱动都内置了对wsl2 cuda的支持，因此我们什么也不用额外装，只需要在wsl里面装上cuda和cuda-tools就好
```sh
sudo pacman -S cuda cuda-tools
```

可能有点大，我这里加一起是7.4g
安装完成后记得source /etc/profile使cuda生效
执行nvidia-smi测试是否安装成功
```
❯ nvidia-smi
Sat Jan 25 13:05:53 2025
+-----------------------------------------------------------------------------------------+
| NVIDIA-SMI 565.77.01              Driver Version: 566.36         CUDA Version: 12.7     |
|-----------------------------------------+------------------------+----------------------+
| GPU  Name                 Persistence-M | Bus-Id          Disp.A | Volatile Uncorr. ECC |
| Fan  Temp   Perf          Pwr:Usage/Cap |           Memory-Usage | GPU-Util  Compute M. |
|                                         |                        |               MIG M. |
|=========================================+========================+======================|
|   0  NVIDIA GeForce RTX 4060 ...    On  |   00000000:01:00.0 Off |                  N/A |
| N/A   43C    P8              1W /   65W |     164MiB /   8188MiB |      0%      Default |
|                                         |                        |                  N/A |
+-----------------------------------------+------------------------+----------------------+

+-----------------------------------------------------------------------------------------+
| Processes:                                                                              |
|  GPU   GI   CI        PID   Type   Process name                              GPU Memory |
|        ID   ID                                                               Usage      |
|=========================================================================================|
|  No running processes found                                                             |
+-----------------------------------------------------------------------------------------+
```

<h2 id="A">FAQ</h2>

Q: GUI应用运行报错，无法连接到wayland display
A: 引自https://github.com/microsoft/WSL/issues/11261#issuecomment-2233443300
执行下列命令
```sh
sudo chmod +rx /mnt/wslg/runtime-dir
ln -s /mnt/wslg/runtime-dir/wayland-0* /run/user/$(id -u)/
```
再次运行程序，应该可以看到窗口
但每次重启wsl都会失效，怎么办呢
因为命令带有sudo，所以不能放在shell初始化里，但我们只需要给它一个别名，就可以啦

Q: 在高分下应用显示很小
A: 引自https://blog.csdn.net/qq_20679687/article/details/128687677
因为wslg还是依靠wayland运行的，所以wayland下可用的缩放方法在wslg下仍然有效
执行以下命令应用缩放比例
```sh
export GDK_DPI_SCALE=1.5 # 1.5就是150%缩放
```

再次启动应用，应该可以看到应用正常缩放
从开始菜单启动的应用因为不走shell默认配置，所以我们要对应用设置环境变量，也就是在快捷方式里添加GDK_DPI_SCALE=1.5

但还有部分应用不遵循这个设置，这个时候就只能看它有没有自带强制缩放功能了，一般说来，只要是以网页渲染为核心的应用，--force-device-scale-factor总是有效的
以chrome为例，它不遵循DPI缩放设置，那就用这个参数
```sh
google-chrome-stable --forcee-device-scale-factor=1.5
```
你完全可以把它做成一个别名，这样下次用就更方便啦.
hmm...我想，美中不足的是，这样缩放的结果是标题栏不会缩放，但标题栏确实作用不大就是啦

Q: 无法运行X11应用，提示cannot open display
A: 引自https://github.com/microsoft/wslg/issues/1192#issuecomment-1980595788
执行以下命令
```sh
ln -s /mnt/wslg/.X11-unix/X0 /tmp/.X11-unix/
```

因为每次开机都会刷新没，所以最好也给它分一个别名

Q: zsh的命令高亮和补全很卡，不跟手
A: 因为微软想增强WSL和win的互通性，所以搞了个WSLENV互通环境变量，导致WSL端的Path非常长，zsh在补全的时候就会查询很长时间的路径，所以会很明显有不跟手的情况
解决办法也很简单，关闭针对Path的互通就可以啦
在/etc/wsl.conf里面添加这一部分
```
[interop]
appendWindowsPath=false
```
之后重启wsl即可生效
但这样的话在wsl里面执行windows程序就有点麻烦了，目前来看最好的解决办法就是选择性添加自己需要的Path，你用到哪些就添加哪些，毕竟你也不会全用到吧

Q: 某些时候会碰到wsl突然卡死，而且开启新终端时提示等待超时
A: 因为复现机会很少，目前尚不清楚是何原因，但解决办法就是重启
（真的只有重启，因为遇到这种问题的时候，wsl是没办法发送关机指令的）

Q: chrome无法启动
A: 如果你看到了有关portal的报错，比如说Failed to read portal version property，那么原因就是，还没安装一个有效的portal后端
从arch wiki来看，portal是为flatpak设计的，沟通沙箱内外资源的框架，但其实任何程序都可以使用这个框架因为chrome全部都是运行在沙箱上的，所以没有这个框架没法正常启动
解决办法也很简单，把portal安装好就可以啦
具体来讲，需要先安装xdg-desktop-portal，然后安装一个后端
xdg-desktop-portal一般都是已经装好了，装一个后端就可以了，这里我以kde的后端为例
```sh
sudo pacman -S xdg-desktop-portal-kde
```

之后就可以啦，chrome应该会正常启动