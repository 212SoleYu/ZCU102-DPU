# Overview

2024.9.20

本文记录了初次接触FPGA工作，在复现江潇师兄的论文过程中实际流程的步骤和很多注意事项，目标是在ZCU102MPSoC上部署DPU，并运行不同的神经网络记录平均推理时间的结果。

虽然师兄的原始实验步骤帮助了我很多，但是由于细节理解不够以及整个xilinx开发套件的版本更迭原因，古早的实验步骤版本不够清楚明了。遂在实验过程中整理出一个新的版本，综合了师兄的版本、我自己的工作日志、AMD的官方文档以及一些论坛查询结果，还有旧版本的DPU部署的官方开发流程文档。

旧的开发流程（以v2.0版本为例）：[Vitis-AI/dsa/DPU-TRD/prj/Vivado at 2.0 · Xilinx/Vitis-AI (github.com)](https://github.com/Xilinx/Vitis-AI/tree/2.0/dsa/DPU-TRD/prj/Vivado)

# Experimental Setup

首先注意版本问题。整个DPU开发流程中，AMD的开发套件之间需要满足版本的匹配。在本实验进行过程中，我选择了参考当前实验室服务器上所部署的vivado版本，为了实现互相匹配，最终选择的工具链版本如下：

- vivado 2023.1 部署在服务器上
- vitis 2023.1 部署在服务器上
- ubuntu 20.04.6 服务器ubuntu版本
- ubuntu 20.04.4 本机虚拟机的ubuntu版本
- petalinux 2023.1 部署在本机的虚拟机上
- DPU IP核等开发套件 v3.5（对于ZCU102来说，3.5与3.0相同）

vivado系列的工具一般版本号统一，例如petalinux2023.1支持vivado 2023.1，相关的ubuntu版本可以在AMD官方文档中查到，例如petalinux2023.1的官方文档ug1144中标注了所支持的ubuntu版本。

由于18系列的ubuntu有点过老，我选择了一个20系列的“中间”版本20.04.4，需要注意的地方在于不要使用sudo apt-get upgrade进行全体升级，会连带内核一起升级。也不要在系统弹出升级版本时点确认，最好系统中关闭询问升级的选项。

vivado和vitis一般安装在服务器上，如果自行安装的话可以参考[Downloads (xilinx.com)](https://www.xilinx.com/support/download/index.html/content/xilinx/en/downloadNav/vivado-design-tools/2024-1.html)寻找对应的版本进行下载即可，选择Web Installer即可，一般会下载+安装5小时左右。注意不要挂梯子下载，小心流量拉爆。

petalinux同样从AMD官网[PetaLinux Tools (amd.com)](https://www.amd.com/zh-cn/products/software/adaptive-socs-and-fpgas/embedded-software/petalinux-sdk.html)中下载文档和资源，下载到虚拟机，petalinux和Borad Support Package BSP文件都要下载。

LTS的Ubuntu可以从各大镜像站的release部分找到，而一些非LTS的老版本的Ubuntu可以从https://old-releases.ubuntu.com/releases/中找到。

实验全程用到的官方文档都可以从[Downloads (xilinx.com)](https://www.xilinx.com/support/download/index.html/content/xilinx/en/downloadNav/vivado-design-tools/2024-1.html)中找到，下载完Vivado也会有DocNav来本地查找对应的开发文档。在实验过程中用到的开发文档编号我会在对应的地方标注。

实验没有真的部署一个vitis-ai，采用的是官方发布的主流神经网络模型的编译好的.xmodel文件，如果需要安装vitis-ai工具同样请参考vitis-ai的github首页。

本文档全程记录vivado流程。使用vitis工作流可能会更简单，详情参考官方文档。

# Petalinux Installation

在本机PC上安装petalinux需要用到的官方文档是UG1144.

petalinux工具是用来给ZCU102的片上控制器配置嵌入式linux系统的工具。

[Installation Requirements • PetaLinux Tools Documentation: Reference Guide (UG1144) • 阅读器 • AMD 技术信息门户网站](https://docs.amd.com/r/BuLNfdhW6a4haNQCGAblDA/VGKtHjwMWQBm6QqWyzQftg)

下载好petalinux和BSP文件后，新的虚拟机先获取sudo权限，此处略过。注意是全程作为non-root用户获取sudo权限后进行安装，不能全程使用root进行。然后将默认的sh由dash修改为bash。如果默认就是dash那么就不需要修改。

使用

```shell
ls -l /bin/sh
```

查看当前默认的sh，使用

```shell
sudo dkpg-reconfigure dash
```

选择no即可修改，可以使用ls命令再次查看。

在下载好的.run 文件目录下使用chmod 755 命令执行该文件，中间会不断告知虚拟机缺少的库，对应安装即可。

需要注意的是在运行给出的信息中不能有This is not a supported OS。如果报了该信息说明下错了版本或者误操作升级了过新的版本。

中途出现的缺失工具可以通过直接使用该命令的方式来让ubuntu给你提示安装命令。还有一些工具或者库不是完全对应的名称，在网上对应查找对应的库即可。

环境检查通过之后可以按照给出的指示同意协议然后安装。可以使用

```shell
./xxx.run --dir <path>
```

来指定安装位置，最好安装在一个空文件夹中，不然会出WARNING，但是也可以继续进行。

过程中会爆出一个关于TFTP服务器的WARNING，暂且按下不表。

安装完成后source setting.sh，使用

```shell
echo  $PETALINUX
```

如果返回了安装路径，那么就是安装成功了。

# Vivado Part

本部分可以参考的官方文档是UG338.

在Vivado部分需要完成的工作是将DPU配置完毕，部署好接线后生成比特流。

但是UG338给出的自定义方式太过底层，建议可以参考着看但是不要全程跟着配置。

首先在服务器或者你自己的本机下载DPU对应的资源套件，截至目前最新的版本下载地址在

[Vitis AI — Vitis™ AI 3.5 documentation (xilinx.github.io)](https://xilinx.github.io/Vitis-AI/3.5/html/index.html)中左侧WORKFLOW AND COMPONENTS部分中DPU IP Details and System Integration部分中找到下载表格，选择reference Design，下载后解压（最好不要下载Only IP）。找到目录中的trd_prj.tcl文件，使用

```shell
vivado -source <dirpath>/trd_prj.tcl
```

如何在服务器上打开vivado需要先找到vivado目录然后source setting.sh。

命令运行后会自动完成接线。如果有高手可以修改甚至自行接线请各凭本事。

validate design。通过后生成比特流，这个过程大概需要两三个小时。如果届时服务器又翻新了，那么vivado可能会存在license的问题，各凭本事解决。

比特流生成后点击file->export hardware, 选择include bitstream。很快会生成一个.xsa文件，将其转移到本机的虚拟机上，进行下一部分的工作。

# Petalinux Compilation

本部分需要的官方文档为UG1144,以及Xilinx Vitis-AI官方的github页面中的DPU IP Details and system Integration部分。[Vitis AI — Vitis™ AI 3.5 documentation (xilinx.github.io)](https://xilinx.github.io/Vitis-AI/3.5/html/index.html)本部分就是实际的根据前面的铺垫配置这个片上的linux系统。

需要先下载前面提到的BSP文件。同样是先source setting.sh来设置环境变量，使用

```shell
petalinux-create -t project -s xxxx.bsp
```

将bsp文件解压，生成一个新的目录,该目录即为petalinux project的目录。

---

下一步参考xilinx的github.io主页中DPU IP Details and system Integration章节中Rebuilding the Linux Image With Petalinux部分。

本部分内容是将vitis-ai-run-time运行库VART添加到可选配置中，好让片上系统可以获得VITISAI的库支持。

网页给了两种方法，第二种没成功，参考第一种。

首先将github仓库中对应的文件夹复制到对应目录下

```shell
cp Vitis-AI/src/petalinux_recipes/recipes-vitis-ai <petalinux project>/project-spec/meta-user/
```

由于本文档采用的是vivado工作流，所以在recipes-vitis-ai下应该有两个.bb文件，二者唯一的区别就是一个工作流的默认配置问题，需要将默认的vitis流程修改为vivado流程。将vart_3.5.bb文件备份后直接注释或者删除文件中的

```shell
#PACKAGECONFIG_append = " vitis"
```

一行即可。然后找到文件/<path of petalinux project>/project-spec/meta-user/conf/user-rootfsconfig，在文件中添加以下三行内容

```shell
CONFIG_vitis-ai-library
CONFIG_vitis-ai-library-dev
CONFIG_vitis-ai-library-dbg
```

添加完毕后保存，在下面一步的用户配置中就能看到这三个选项了。

---

接下来是一个比较重要的部分，关于内核文件部分的用户自定义配置，也就是petalinux-config命令。实验的失败次数大部分出自此处。

根据UG1144的附录K中描述，petalinux-config有-c和--get-hw-description两个独立且必须的配置步骤。

先配置--get-hw-description选项。

```shell
petalinux-config --get-hw-description <path> --silentconfig
```

进行编译。这里的path是.xsa文件的位置。silentconfig参数表示不呼出可视化配置页面并且保留此前配置。

下一步使用petalinux-config -c选项对于用户系统进行配置。根据UG1144描述，-c选项共有三个可选项：

- petalinux-config -c kernel 配置内核选项
- petalinux-config -c u-boot 配置启动选项
- petalinux-config -c rootfs 配置用户文件系统

流程中用到了kernel选项和rootfs选项。注意都不要添加--silentconfig参数，要呼出配置窗口进行配置。此处有参考博客还配置了内存大小、启动地址等参数，我在流程中没有做出修改。

使用petalinux-config -c kernel启动对内核的配置，初次调用该命令会编译半个多小时，后面再启动就会快一点。

呼出配置窗口后搜索DPU，可以找到在Device Drivers -> Misc devices->Xilinx Deep learning Precessing Unit (DPU) Driver。按理来说这个应该是默认为勾选状态的，但是在本实验流程中默认是非勾选，导致了很多失败。勾选后一定保存然后退出。如果不配置这一点，会导致在进入linux系统后调用dpu(show_dpu 命令)显示找不到可用的实例，具体的报错信息为：

```shell
 failed: !get_factory_methods().empty() 
 Aborted
```

搜索中的另一个DPU相关驱动我没有勾选，从结果来看似乎是没有直接影响。

完成内核配置后使用petalinux-config -c rootfs对rootfs区进行配置。

注意，如果这里不使用petalinux-config -c配置包，只使用--get-hw-description参数，也是可以正常生成一个镜像插入SD卡启动的，但是这时候生成的petalinux镜像在片上是一个裸机，没有python，没有基础的软件管理工具，甚至没有make工具，所以是无法使用的（可能有更加底层的办法，能力有限掌握不了）。

```shell
petalinux-config -c rootfs
```

呼出用户的自定义选项环境，在Image Feature选项中勾选package-management，下方的不需勾选。（详情参考UG114中Chapter12的Package Management部分）.有了包管理工具就可以在片上按需安装所需要的工具了。

在user-packages部分勾选vitis-ai-libaray，如果在前面一步没有配置recipes这里就没有vitis-ai可选。-dev和-dbg后缀分别是开发和调试版本，选择默认的无后缀即可。

在Filesystem Packages部分，用户可以自定的勾选若干软件进行安装，当然也可以进系统使用dnf进行安装。

推荐使用Petalinux Package Groups挑选必须调用的库的群组安装即可，当前版本的packagegroup包含内容参考[PetaLinux Package Groups - 2023.1 Release - Xilinx Wiki - Confluence (atlassian.net)](https://xilinx-wiki.atlassian.net/wiki/spaces/A/pages/2615508993/PetaLinux+Package+Groups+-+2023.1+Release)。实验中本人用到的群组有：

- 需要python所以添加packagegroup-petalinux-python-modules，
- 需要基础gcc相关所以选择packagegroup-newlib-standalone-sdk-target，（这个packagegroup其实在Filesystem Packages->misc->packagegroup-core-standalone-sdk-target）
- 需要opencv所以可以添加packagegroup-petalinux-opencv
- 需要基础的命令例如cmake所以添加packagegroup-petalinux

选完必要的环境后记得保存。

----

配置完成软硬件环境后然后使用petalinux-build命令即可，等待安装的过程中我一共出过两次报错，一次是选的包太多导致镜像大小溢出，解决方案是减少软件包的勾选；一次是url找不到下载源，解决办法是挂梯子，或者将对应的库取消勾选（如果不重要的话）。

```shell
petalinux-build
```

过程中可能会报错显示缺少某个库，对应安装即可。编译的过程平均不到一个小时。20-30分钟。

```shell
cd images/linux
petalinux-package --boot --fsbl zynqmp_fsbl.elf --u-boot u-boot.elf --pmufw pmufw.elf --fpga system.bit
```

最后在打包这一步，按照上述命令安装即可。

需要注意的是，此前试验的过程中遇到过没有system.bit的情况，如果发生了建议重新进行前面的步骤排查问题。但是如果system.bit文件是正常生成的情况下，其可能是不会在图形化界面下展示的，使用ls命令是可以正常看到的，此处可以注意一下。

这个petalinux-build过程在我的电脑中两小时内就完成，也无需挂梯子。师兄的原步骤中说这里既需要vpn又慢，还给出了解决方案。我没用到，仅供参考。

https://cloud.tencent.com/developer/article/1663482?from=article.detail.1757063

# SD Card

配置SD卡的部分参考UG1144中的附录I，但是要比UG中的操作多一些。

本部分是将一张SD卡配置为petalinux系统盘，插入到ZCU102中用作启动。包括一个boot分区1和一个rootfs分区2.

使用读卡器将一张SD卡连接到本机的虚拟机（petalinuxtool所在的机器）。如果是使用vbox连接不到USB可以尝试在全局设定中调整USB协议从2.0到3.0.

如果SD卡是空的，那么可以跳过到下一部分；如果不是空的，那么我们需要先将原有的SD卡格式化并删除分区重新配置。

----

首先在SD卡连接到虚拟机之后使用

```shell
sudo fdisk -l
```

命令可以查看所有的可用的磁盘（并非已挂载的磁盘），使用

```shell
df -h
```

命令可以查看已经挂载了的磁盘。

查看SD卡的挂载点，一般来说虚拟机都会挂在/dev/sdb。使用

```shell
sudo umount /dev/sdb1
sudo umount /dev/sdb2

sudo fdisk /dev/sdb
```

取消挂载并进入磁盘管理工具，按下m获取参数帮助。使用d命令将分区删除。

如果是非空的，则需要使用d命令将每个分区全部删除，删除后使用w保存。重新使用

```shell
sudo fdisk -l
```

进行时查看到原有的分区不见为准。

----

如果SD卡是空的或者已经完成了删除工作，则先将其解除挂载，然后使用

```shell
sudo fdisk /dev/sdb
```

进入管理工具，使用n创建新的分区，模式和字节大小参考ug1144附录中的数据。使用t命令将第一个分区的文件类型设置为FAT32，代号为c。

配置完成后记得使用w进行保存，另外需要注意使用a命令将boot分区标注为boot，查看时会有星号标识。

最终查看时应当为两个分区，第一个分区有星号，type为FAT32，第二个分区的type为linux或EXT4.

配置好分区的种类之后，使用以下两个命令将分区进行格式化

```shell
sudo mkfs.vfat /dev/sdb1
sudo mkfs.ext4 /dev/sdb2
```

完成格式化之后，下一步将这个SD卡的两个分区挂载到linux的文件系统中，通过以下命令：

```shell
sudo mkdir /media/yuyunlong/boot
sudo mount /dev/sdb1/boot

sudo mkdir /media/yuyunlong/rootfs
sudo mount /dev/sdb2/rootfs
```

完成挂载之后，可以通过df-h命令再次确认挂载成功。

---

再下一步，需要特别注意！！！！！！

再UG1144中，文档显示在完成了格式化和挂载后应当向boot分区复制4个文件，但是这4个文件其中有一个是错的！

根据实际操作情况，应该向boot分区中复制 **BOOT.BIN, boot.scr,image.ub** 和 **system.dtb**,官方文档v2023.1中的**Image**是不对的！（我这样说是基于实际结果的，如果最新版本的UG中做出了修改请结合官方文档和自己的实际结果为准。）

另外，还需要向rootfs分区中解压rootfs.tar.gz文件，此处应该使用命令

```shell
sudo tar -zxvf rootfs.tar.gz -C /media/yuyunlong/rootfs
```

注意tar命令中的z选项不能缺少，实验中有一次可能因为z选项的缺少导致了失败。

---

接下来的步骤是师兄文档中的，我暂时没在官方文档中找到这一部分，但是还是全部操作了一遍。

复制和解压完成后使用sync命令进行同步，否则会报错

再进行一次权限调整：

```shell
$ sudo chown root:root /media/yuyunlong/rootfs/
$ sudo chmod 755 /media/yuyunlong/rootfs/
```

注：对应修改目录名称即可。

完成后安全退出SD卡。

# ZCU102 Borad Connection

实际连接ZCU102。

首先将SD卡插入卡槽，连接电源线到电源，J83端口对应的UART模块的USB线到PC的USB接口，还需要将主机、ZCU102通过网线连接到一个路由器。如果不通过这种方式可能需要配置IP等操作，我暂时没用到，可以参考别人的博客。

调整ZCU102上的SW6，调整至SD卡启动的模式，1开关在上，234开关在下（以开关上的字正视为参考方向）。

其中，J83接口是USB转UART的协议接口，需要在主机上安装一个驱动，从而能在设备管理器中找到Silicon Labs Quad CP2108 USB to UART Bridge：Interface 0-3.

在[CP210x USB to UART Bridge VCP Drivers - Silicon Labs (silabs.com)](https://www.silabs.com/developers/usb-to-uart-bridge-vcp-drivers?tab=downloads)官网下载驱动并且安装即可。

使用MobaXterm打开session中的serial，任选四个Silicon接口中的一个，波特率选择115200（默认）建立连接，打开ZCU102的电源开关，串口中会发送大量的消息。如果不显示消息或者显示Press ESC to Enter Controller Mode，可以退回上一步检查SD卡是否没有正确设置boot分区。

成功的话，会显示要求你进行登录login，此时输入用户名petalinux，他会显示你改密码，修改完毕后就保存了该账户。我设置了用户名和账号同为petalinux。

进入命令行后，就是在片上搭载了一个linux系统，使用ifconfig查看ip地址，我的IP地址为

```
192.168.50.199
fe80::20a:35ff:fe05:2da5/64
```

在使用MobaXterm切换SSH连接上述IP地址，登录即可在ssh中直接使用命令行操作片上的linux系统。

# On-Board Execution

在Petalinux Compilation部分至少配置了dnf包管理系统之后，此时应该至少在片上的linux系统中有了一个可以管理包的工具。我的程序是直接安装好了python，但是缺少运行代码所需要的库。因此直接使用命令行安装对应的库即可。

如果至此配置顺利的话，使用命令

```shell
sudo show_dpu
```

就可以输出DPU内核的消息了，数量是与Vivado中配置的数量是一致的，例如我配置了三个，输出结果就应该是

```
dpu_kernel_0
dpu_kernel_1
dpu_kernel_2
```

 如果不使用sudo 会导致权限问题打不开/dev/dpu，但是也就只是一个权限问题。

将调度所用的python代码，需要用到的.xmodel文件传入，并给他配上训练的数据集，就可以开始运行了。

如果在运行过程中出现了Fingerprint，说明.xmodel文件出现了不匹配的情况。建议从github Xilinx vitis-ai model-zoo中下载。

# Summary

本文档仅用作记录工作流程和记录配置中的坑，如果有未尽事宜以Xilinx或者其他软硬件设计的官方文档为准。

2024.9.25