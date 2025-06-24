---
Title: Windows下WSL使用简介
Authors: 王程玄
comments: true
---
# Windows下WSL使用简介

## 前言：为什么需要WSL

由于种种原因，组内提供的工作电脑的操作系统已经逐渐从Mac转向Windows（更严谨的应该说是x86_64架构的组装机），同时近年来Windows笔记本端可选择性更为丰富，综合诸多因素来看在一些体验上已经可以做到不弱于Mac。而由于Windows操作系统并不是基于Unix的系统，因此系统自带的终端操作逻辑以及文件系统和liunx本身有较大的区别，在使用时可能会带来一些困惑以及不兼容的情况。而额外学CMD命令有些得不偿失而且可能会和linux终端命令搞混来。

而对于Windows下较为成熟的各种终端软件如MobaXTerm，Putty，他们在使用体验上远远不及Mac系统本身带的终端，且整体逻辑较为 **割裂** ，无法将系统和终端直接联系起来。如在MobaXTerm下使用其直接传递文件的速度并不快，大多时候限制因素并不是网速，远不如使用`scp`命令进行的文件传；Putty本身不支持x11，如果需要配置远程可视化窗口流程复杂，而且需要在使用时挂一个x11转发软件，同时不支持`scp`本地文件传递，想要传个文件还需要再开一个windows终端或者scp软件，这太不优雅了。

而WSL（Windows Subsystem for Linux）的出现大大缓解了这种令人两难的情况，单纯从使用者的角度来看，其和直接使用linux命令对Windows进行管理没有太大的区别，我们可以同时享受到Windows可视化界面在信息处理方面带来的便利以及linux命令行带来的批处理方面的便利性。特别是在ssh的使用方面，我们可以将服务器的集群操作以及文件传输同时在一个终端内进行体验较为良好的完成，这在之前的任何软件中都是没有很好地实现的。

如果电脑本地有一些性能，我们还可以尝试在wsl中安装一些计算软件或者调试环境，这样可以在本地做一些短期的参数验证以及代码调试，不需要反复提交到集群上进行排队，提高一定效率的同时可以节省一些机时费用。如cp2k中的`REL_CUTOFF`以及`CUTOFF`参数的调优可以在本地快速进行，再在集群上进行长时间的计算。

## WSL安装

此过程较为容易，且[官方教程](https://learn.microsoft.com/zh-cn/windows/wsl/install)、中文教程在互联网上较为详尽，不再过多地占篇幅。

从笔者的使用经验来看，Windows Terminal是一个比较不错的终端（Win11系统默认，Win10需要自己安装），在新版中使用wsl2可以直接支持x11的转发，不行的可以参考[网页](https://blog.dengqi.org/posts/%E4%BD%BF%E7%94%A8-windows-%E8%87%AA%E5%B8%A6ssh%E7%9A%84x11%E8%BD%AC%E5%8F%91%E5%8A%9F%E8%83%BD%E5%B9%B6%E9%85%8D%E7%BD%AEssh%E5%92%8Cvscode/)进行设置。

!!! info "版本推荐"        
    目前wsl默认版本为wsl2，可以完整支持linux内核的功能（最大的进化是支持了docker和systemd），后面的内容默认以此版本为准。同时linux版本也可以选择默认的Ubuntu，作为在个人端应用较广的linux发行版，在遇到问题时通过互联网查找信息能够解决的概率相对较大。

终止/重启wsl特定版本的方法，在Powershell/CMD中：
```PowerShell
wsl --shutdown <distro_name>
distro_name    # 启动指定发行版
```
其中`distro_name`为安装的发行版名称，可以通过`wsl -l -v`查看已经安装的发行版。

当然，WSL有着极强的自我管理意识，在WSL中也可以直接实现自闭和重启，这得益于WSL可以直接调用windows下面的程序，只要`wsl.exe`在windows的环境变量当中，就可以实现在wsl中的直接调用：
`wsl.exe --shutdown <distro_name>`，然后直接在终端下按回车就可以直接实现重启。

### 将WSL移动到其他盘符

wsl默认安装在C盘下，如果C盘初始分配的相对较小或者有C盘空间焦虑，可以考虑将指定的wsl移动到其他的盘符下。请参考：[轻松搬迁！](https://zhuanlan.zhihu.com/p/621873601)

## WSL配置ssh环境

在基础设置上和[ssh使用](./ssh_note.md)章节没有区别。下面将介绍一些便利使用的方法。

### 改善使用体验的关键：WSL的vpn环境

对于在非校园网环境下办公的情况，wsl和EasyConnect的兼容性并不是很好，经常出现开启EasyConnect之后wsl无法访问网络的情况。如果想要正常使用两者，就需要在开机后迅速打开EasyConnect客户端，再打开wsl，但是这种连接也非常脆弱，中间但凡EasyConnect断开，就再也别想连回去了，唯一解决方法就是重启，这无疑会为工作带来相当的困扰。为此，有以下的几种通过环境配置的解决方法，其中最为推荐的是第一种方法：

#### 1. 使用wsl-vpnkit同步Windows的网络环境

工具地址：[wsl-vpnkit](https://github.com/sakai135/wsl-vpnkit)

在使用上，最无感的安装方式为使用wsl的systemd方法建立服务，这种方法不需要建立新的发行版也不需要每次开机后都重启动，操作过程如下：

首先需要获取wsl-vpnkit程序本体
```bash
# 安装依赖
$ sudo apt-get install iproute2 iptables iputils-ping dnsutils wget

# 下载wsl-vpnkit并解压其中关键组件
$ VERSION=v0.4.x    # 目前最新版本为v0.4.1
$ wget https://github.com/sakai135/wsl-vpnkit/releases/download/$VERSION/wsl-vpnkit.tar.gz    # 可能需要科学上网下载再传到wsl中
$ tar --strip-components=1 -xf wsl-vpnkit.tar.gz \
    app/wsl-vpnkit \
    app/wsl-gvproxy.exe \
    app/wsl-vm \
    app/wsl-vpnkit.service
$ rm wsl-vpnkit.tar.gz
```
此后在路径下就可以看到`wsl-vpnkit, wsl-gvproxy.exe, wsl-vm, wsl-vpnkit.service`四个文件，编辑其中的`wsl-vpnkit.service`文件，以下为该种配置方式的参考：
```bash
$ vi wsl-vpnkit.service

[Unit]
Description=wsl-vpnkit
After=network.target

[Service]
# for wsl-vpnkit setup as a distro
#ExecStart=/mnt/c/Windows/system32/wsl.exe -d wsl-vpnkit --cd /app wsl-vpnkit

# for wsl-vpnkit setup as a standalone script
ExecStart=/usr/local/bin/wsl-vpnkit/wsl-vpnkit
Environment=VMEXEC_PATH=/usr/local/bin/wsl-vpnkit/wsl-vm GVPROXY_PATH=/usr/local/bin/wsl-vpnkit/wsl-gvproxy.exe

Restart=always
KillMode=mixed

[Install]
WantedBy=multi-user.target
```
此后需要将运行文件和服务配置文件放到该放的位置：
```bash
$ sudo mkdir /usr/local/bin/wsl-vpnkit
$ sudo mv wsl-vpnkit wsl-gvproxy.exe wsl-vm /usr/local/bin/wsl-vpnkit/
$ sudo mv wsl-vpnkit.service /etc/systemd/system/
```
其中`/usr/local/bin/wsl-vpnkit`可以是任意位置，确保wsl开机后不会消失就可以。此后就可以启动服务并设置开机启动：
```bash
$ sudo systemctl enable wsl-vpnkit    # 设置开机启动
$ sudo systemctl start wsl-vpnkit     # 开启服务
$ systemctl status wsl-vpnkit         # 查看运行状态
```
如果没有查看状态没有报错，就说明已经可以正常运行了。

*其他的运行模式可以参考[github界面](https://github.com/sakai135/wsl-vpnkit?tab=readme-ov-file#setup-as-a-distro)进行设置。*

#### 2. 设置WSL的网络模式为mirrored

!!! warning "注意"  
    网络映射在wsl新版本中已经转为正式功能，但是实测使用EasyConnect仍然会出现莫名其妙的卡顿，使用体验较为糟糕。同时旧版本wsl无法使用。
    另：如果想要在WSL中使用ipv6，使用mirrored是一个非常好的选择。

wsl默认的网络模式为NAT，而EasyConnect的工作原理可以简单的理解为建立了一个虚拟网卡和校园网进行沟通，这两种方式并不能进行完美的配合，具有很强的先后运行顺序的关联。而在新版的wsl版本中，提供了一种镜像Windows中所有的网络环境到wsl中的手段，包括系统代理也可以镜像到wsl当中。

首先设置`.wslconfig`文件：
```bash
$ vi /mnt/c/Users/<windows_user>/.wslconfig

[wsl]
networkingMode=mirrored
```

其中`/mnt/c/Users/<windows_user>/.wslconfig`是Windows下的用户根目录下的一个wsl配置文件，一般是新创建的。同时，该文件也可以设置wsl占用的系统资源等一些参数。完成编辑保存并退出后，重启wsl发行版就可以实现网卡的映射。可以通过以下的命令进行观察:
```bash
$ sudo apt install net-tools
$ ifconfig
```

可以将输出的信息和任务管理器中的网卡信息进行对比，可以发现任务管理器中的网卡ip已经映射到wsl当中，我们也可以访问校园网服务器。

#### 3. WSL2 + Docker + EasyConnect + Clash

参考：[新时代的快乐科研：WSL2+Docker+EasyConnect+Clash](https://cloudac7.github.io/p/%E6%96%B0%E6%97%B6%E4%BB%A3%E7%9A%84%E5%BF%AB%E4%B9%90%E7%A7%91%E7%A0%94wsl2-docker-easyconnect-clash/)

该方式的实现非常优雅，并且因为容器化的原因能够让EasyConnect非常老实，且可以登录一个vpn让多个设备使用，让校园网外的手机和平板平台也能访问校园网（移动端的EasyConnect并不支持扫码登录）。比较麻烦的点在于如果你的Clash配置文件会自动更新，会覆盖自己加入的那一行转发规则，~~不更新可能导致和某scholar检索say byebye~~。同时`.ssh/config`文件的编写会稍微有一些问题，如果在校园网下工作没有打开vpn的情况是不需要走代理端口的，此时若不修改文件内容就会报错（

### WSL的文件系统: scp使用相关

此处是WSL相对于其他解决方案最有优势的地方。WSL可以将Windows的硬盘挂载在文件系统上，我们可以直接通过`df -h`命令进行查看：
```bash
$ df -h
Filesystem      Size  Used Avail Use% Mounted on
...
/dev/sdc       1007G   50G  907G   6% /
...
C:\             203G   88G  115G  44% /mnt/c
D:\             196G   80G  116G  41% /mnt/d
E:\             528G   52G  476G  10% /mnt/e
```
在此处，硬盘被分了三个盘符，均可以直接在WSL中看到，其默认的挂载点是`/mnt/x`，并可以直接通过WSL下的linux命令进行操作，如我想将桌面上的所有python脚本传递到集群上：
```bash
$ scp /mnt/disk/Users/<windows_user>/Desktop/*py server:~/scripts
```
其中`disk`为电脑桌面的所在的盘符（如果没有移动过默认在C盘，则其值为c），`<windows_user>`为你的电脑用户名（这也是不建议将用户名设置为中文的原因之一，可能会引起错误或者输入不便），`server`可以通过`.ssh/config`进行设置。

每次都需要输入这么一长串路径会很麻烦，因为挂载点在启动时并不会改变，因此可以将常用的文件夹映射到`$HOME`下：
```bash
$ ln -sf /mnt/disk/Users/<windows_user>/Desktop $HOME/windows_desktop
```
因为建立的是软链接，因此在Windows中做的改动都会反映在文件当中，不需要反复进行同步。此后就可以直接将`$HOME/windows_desktop`中的文件通过`scp`传递到服务器上了。

#### 挂载额外磁盘

如何创建虚拟磁盘请参考如何创建hyper-v虚拟机磁盘。

目前WSL并不支持自定义自动挂载额外虚拟硬盘，但是可以手动挂载磁盘，对于在windows路径`<route_to_disk>`下的虚拟磁盘，其最简单的方式为

```PowerShell
wsl --mount --vhd <route_to_disk> --bare
```

然而无论如何挂载，在WSL重启后都需要进行重新挂载，我们可以用一个简单的bat脚本来使得每次重启WSL后都能自动挂载

```
C:\wsl_automount.bat(or somewhere u like)

wsl --mount --vhd <route_to_disk> --bare && wsl -d <distro_name>

C:\Users\<windows_user>\AppData\Roaming\Microsoft\Windows\Start Menu\Programs\Startup\wsl_startup.vbs

set ws=wscript.CreateObject("wscript.shell")
ws.run "cmd.exe /c C:\wsl_automount.bat", 0
```

这样可以将`wsl_startup.vbs`脚本放在启动文件夹下，就可以在Windows启动的时候自动启动WSL，同时在终端退出后也不会自动关闭WSL，避免一些后台任务被中止。

此后就可以在WSL中的`/etc/fstab`中规定虚拟磁盘的挂载位置（最好使用uuid的方式指定磁盘），或者在`/etc/wsl.conf`中的`[automount]`选项中更改默认的挂载位置，但是这项的更改同样会更改Windows物理磁盘的挂载选项。

### `.ssh/config`文件同步：VSCode使用一致性*

如果想要在Windows系统中使用VSCode连接远程服务器，就要调整Windows路径下的`.ssh/config`文件进行调整，然而我们平常的终端工作往往在WSL系统中进行，导致两套`.ssh/config`文件不同，这也是造成使用感割裂的原因之一。为了解决这个问题，我们可以考虑将Windows下的`.ssh/config`软链接至`~/.ssh/config`，这样修改时就是同步的。

然而在WSL的默认模式中，`/mnt/c`的权限为777，而这和`config`文件要求的权限差的很多（`config`文件要求600），而在默认的挂载方式下，并不能修改文件的权限，因此需要调整默认的挂载选项，在`/etc/wsl.conf`中加入：
```bash
[automount]
options = "metadata,uid=1000,gid=1000,umask=002,fmask=002,dmask=002"
```

其中`uid`,`gid`可以通过`id`命令获取，`*mask`使用方法可以自行查阅。修改完成后重启发行版，发现可以修改挂载的文件的权限了：
```bash
chmod 600 /mnt/c/Users/<windows_user>/.ssh/config
mv ~/.ssh/config ~/.ssh/config.bak
ln -sf /mnt/c/Users/<windows_user>/.ssh/config ~/.ssh/config
```

当然，我们也可以将常用需要使用VSCode的服务器单独列一个`config`文件并在VSCode的设置中进行修改。

## Docker Desktop：简单配置计算软件的利器

如果觉得自己手动配置具体的计算软件太麻烦，或者配置的时候遇到了奇奇怪怪的bug，且在本地运行只是一些短运行的简单的参数检查的需求，不需要极致运行效率，就可以尝试使用容器化的方法对想要实现的功能进行快速的部署。而在WSL中运行docker也很方便，只需要下载一个Docker Desktop，在安装的时候把涉及WSL及其发行版的选项勾选上，启动之后就能够在WSL中调用docker的相关功能（这也是WSL2的优势之处）。

具体的安装过程就不再赘述，这里备注几个可以优化docker使用体验的关键词：dockerhub换源、移动docker虚拟磁盘位置、`docker login -u <username>`而不是在Docker Desktop当中点击登录（developer）。

这里使用一个简单的[cp2k](https://github.com/cp2k/cp2k-containers)例子（也是编译相对复杂、依赖多的计算软件）对docker的基础使用进行说明：
```bash
docker run -it --rm --shm-size=12g -v $PWD:/mnt -u $(id -u $USER):$(id -g $USER) cp2k/cp2k:2025.1_openmpi_generic_psmp mpirun -bind-to none -np 9 -x OMP_NUM_THREADS=2 cp2k -i input-128.inp
```
在运行这条命令之后，会从镜像源下载镜像（这个例子中大概4G）。简要说明下这行命令当中的一些参数：
- `--shm-size=12g`: 这个是linux中将内存当作硬盘的区域，可以显著提高计算过程中的文件读写速度。在默认设置中通常为整个内存的一半，而在容器当中的默认设置很小，这里需要根据内存大小手动设置一下，一般设为4g以上对于计算速度的影响就比较小了。
- `-v $PWD:/mnt`: 将当前的目录挂载到容器`/mnt`目录下，cp2k的镜像默认在这个路径下进行计算。所以需要将输入文件中的所有内容都放到当前目录下，主要包括基组文件和色散文件（当然也可以利用`-v`手动挂载到一个地方，对容器比较熟悉的同学可以尝试）。并在input文件中使用相对路径（建议）或者自己转为`/mnt`的绝对路径进行指定。
- `cp2k/cp2k:2025.1_openmpi_generic_psmp`: 这里制定使用镜像的名称和版本，可以根据自己的需求进行调整，具体的版本可以在docker hub上查询。
- `mpirun -bind-to none -np 9 -x OMP_NUM_THREADS=2`: 这些是mpi运行的相关参数，其中`-x OMP_NUM_THREADS=2`一定要这样设置OpenMP的使用的线程数，直接设置为环境变量不会起作用。如果用的是`mpich`版本的会略有不同，详见上面的github页面。
- `cp2k -i input-128.inp`: 前面将本地路径挂载到了容器当中，这里就直接使用本地路径下的input文件名称即可。

其他的计算软件如`deepmd-kit`也有镜像版本，可以很方便地运行势函数的训练和推理，在不想折腾显卡libtorch环境的时候（特别是dp3.0版本）也不妨为一种好的选择。

容器化的另外一个优势在于能够在架构相同的不同的平台上使用相同的容器运行，目前嘉庚集群和chenggroup集群均安装了`singularity`作为容器化的平台，在使用上和docker大同小异，并且可以将docker的镜像直接转为对应的可执行文件(?)，如有复杂的软件部署需求，可以考虑在本地构建镜像传到集群上运行。

## WSL软件安装踩坑（未完待续）

WSL发行版：Ubuntu-24.04

### CP2K
不要使用`sudo apt install mpich`的方法安装mpi运行库，这会导致即使使用`psmp`版本也无法使用mpi并行计算的bug，`mpirun`会导致同时运行n个相同的子程序。如果不想自己编译mpi库的话，因为不是集群多软件环境一般没有依赖冲突，使用`--with-openmpi=install`，此后使用时`source /dir/to/cp2k/tools/toolchain/setup`设置环境变量即可。

### VMD
在使用WSL2安装linux版本的vmd的时候，如果使用默认的OpenGL渲染器，会出现渲染速度过慢，导致轨迹预览不流畅的问题出现。而WSL2的文件系统性能相对于WSL1的模式较差，网络上提供的使用WSL访问windows下的vmd程序的方法会遇到加载不出轨迹文件的问题。

解决方法：
- 一种解决方法是将默认的OpenGL渲染器换为OpenGL-GLSL渲染器。如果想改变默认的渲染，可以将`display rendermode GLSL`加入到`~/.vmdrc`文件当中，这样vmd预览轨迹就会变得流畅，不再受到渲染器性能的影响。一般来说这种方法对性能的影响相对较小。

- 比较通用的解决方法是更换默认的渲染器，比如较为通用的`llvm`渲染器，直接使用CPU对画面进行渲染，调用方法为在`.bashrc`中加入以下语句或者每次运行前输入下
```bash
export LIBGL_ALWAYS_INDIRECT=0 
export GALLIUM_DRIVER=llvmpipe
```
这样的方法可以统一将所有的渲染器改为`llvm`，包括`ovito`等可视化软件的渲染问题同样可以解决。问题在于比较吃CPU性能，显卡的渲染优势无法完全利用上。

### PING
因为使用[wsl-vpnkit](https://github.com/sakai135/wsl-vpnkit/issues/254)，当使用WSL原生的ping命令测试远程主机的连通性时，会发现对于内网主机无论如何都能ping通，但是使用windows的Powershell则不会出现如此的情况。
目前暂时没有找到具体的原因，在排查网络问题时可能会带来一定的困扰。可以使用windows的PING.EXE代替WSL下的ping：`alias ping="/mnt/c/Windows/system32/PING.EXE -t"`，这样默认调用的ping就是windows下的，反应的网络信息会更准确。

### IPV6
WSL默认的网络模式`NAT`并不支持ipv6的相关功能，使用`mirrored`模式可以解决，但是对于SSLVPN实在是太卡了！！！~~（没关系306也不支持ipv6说是）~~
