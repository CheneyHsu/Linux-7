## PXE 自动化安装

### 解决问题和注意事项：
其实前面讲过了自定义光盘安装，但是光盘也有一定的弊端的，因为自定义光盘还是需要很多人工干预，而这种PXE网络安装的模式可以更省力，只要主机上架，然后开机，选择安装所需系统，然后远程连接配置系统，既可以完工了。
虽然PXE安装可以减少更多的工作量，但是这样PXE在生产中使用需要特别小心，并且也有一定弊端，试想一下，你的整个生产网络是可以互通的，在你开放PXE安装其它主机系统的时候，碰巧有一台生产主机重启了，并且开机的引导方式是网络引导启动，这个时候就完蛋了，该系统会被重新PXE一次，所以，要使用这种技术做网络安装，前提一定要做好如下几点：

    1：生产所有主机的引导全部置于硬盘引导
    2：PXE安装服务器对所需要安装的系统要做限制
    3：生产所有服务器的IP地址必须置为静态，切勿使用DHCP自动获取

### 技术点 kickstart + pxe
#### 什么是Kickstart

KickStart是一种无人职守安装方式。KickStart的工作原理是通过记录典型的安装过程中所需人工干预填写的各种参数，并生成一个名为ks.cfg的文件；在其后的安装过程中(不只局限于生成KickStart安装文件的机器)当出现要求填写参数的情况时，安装程序会首先去查找KickStart生成的文件，当找到合适的参数时，就采用找到的参数，当没有找到合适的参数时，才需要安装者手工干预。这样，如果KickStart文件涵盖了安装过程中出现的所有需要填写的参数时，安装者完全可以只告诉安装程序从何处取ks.cfg文件，然后去忙自己的事情。等安装完毕，安装程序会根据ks.cfg中设置的重启选项来重启系统，并结束安装。

#### 什么是PXE

严格来说，PXE 并不是一种安装方式，而是一种引导的方式。进行 PXE 安装的必要条件是要安装的计算机中包含一个 PXE 支持的网卡(NIC)，即网卡中必须要有 PXE Client。PXE (Pre-boot Execution Environment)协议使计算机可以通过网络启动。协议分为 client 和 server 端，PXE client 在网卡的 ROM 中，当计算机引导时，BIOS 把 PXE client 调入内存执行，由 PXE client 将放置在远端的文件通过网络下载到本地运行。运行 PXE 协议需要设置 DHCP 服务器和 TFTP 服务器。DHCP 服务器用来给 PXE client(将要安装系统的主机)分配一个 IP 地址，由于是给 PXE client 分配 IP 地址，所以在配置 DHCP 服务器时需要增加相应的 PXE 设置。此外，在 PXE client 的 ROM 中，已经存在了 TFTP Client。PXE Client 通过 TFTP 协议到 TFTP Server 上下载所需的文件。

#### PXE 工作流程
![jpg](./images/pxe/pxe-1.jpg)

1. PXE Client向DHCP发送请求

首先，将支持PXE的网络接口卡（NIC）的客户端的BIOS设置成为网络启动，通过PXE BootROM（自启动芯片）会以UDP（简单用户数据报协议）发送一个广播请求，向网络中的DHCP服务器索取IP地址等信息。

2. DHCP服务器提供信息

DHCP服务器收到客户端的请求，验证是否来至合法的PXE Client的请求，验证通过它将给客户端一个“提供”响应，这个“提供”响应中包含了为客户端分配的IP地址、pxelinux启动程序（TFTP）位置，以及配置文件所在位置。

3. PXE客户端请求下载启动文件

客户端收到服务器的“回应”后，会回应一个帧，以请求传送启动所需文件。这些启动文件包括：pxelinux.0、pxelinux.cfg/default、vmlinuz、initrd.img等文件。

4. Boot Server响应客户端请求并传送文件

当服务器收到客户端的请求后，他们之间之后将有更多的信息在客户端与服务器之间作应答, 用以决定启动参数。BootROM 由 TFTP 通讯协议从Boot Server下载启动安装程序所必须的文件（pxelinux.0、pxelinux.cfg/default）。default文件下载完成后，会根据该文件中定义的引导顺序，启动Linux安装程序的引导内核。

5. 请求下载自动应答文件

客户端通过pxelinux.cfg/default文件成功的引导Linux安装内核后，安装程序首先必须确定你通过什么安装介质来安装linux，如果是通过网络安装（NFS, FTP, HTTP），则会在这个时候初始化网络，并定位安装源位置。或许你会说，刚才PXE不是已经获取过IP地址了吗？为什么现在还需要一次？这是由于PXE获取的是安装用的内核以及安装程序等，而安装程序要获取的是安装系统所需的二进制包以及配置文件。由于它们需要的内容不同造成PXE模块和安装程序是相对独立的，PXE的网络配置并不能传递给安装程序。从而进行两次获取IP地址过程。

接着会读取该文件中指定的自动应答文件ks.cfg所在位置，根据该位置请求下载该文件。

6. 客户端安装操作系统

将ks.cfg文件下载回来后，通过该文件找到OS Server，并按照该文件的配置请求下载安装过程需要的软件包。

OS Server和客户端建立连接后，将开始传输软件包，客户端将开始安装操作系统。安装完成后，将提示重新引导计算机。这个时候注意，在重新引导的过程中一定要将BIOS修改回从硬盘启动，不然的话又会重复的自动安装操作系统。

在上面介绍中PXE client是需要安装Linux的计算机，TFTP Server、DHCP Server和NFS Server运行在另外一台Linux Server上。Bootstrap文件、配置文件、Linux内核都放置在Linux Server上TFTP服务器的根目录下。而Linux根文件系统存放于NFS Server的共享目录中。

PXE client在工作过程中，需要三个二进制文件：bootstrap、Linux 内核和Linux根文件系统。Bootstrap文件是可执行程序，它向用户提供简单的控制界面，并根据用户的选择，下载合适的Linux内核以及Linux根文件系统。 

### PXE无人值守安装配置

#### Dnsmasq
1.	配置主机静态IP地址 （实验主机：192.168.13.132）

2.	安装dnsmasq

    yum -y install dnsmasq

3.	配置文件在/etc/dnsmasq.conf，做为一个工程师在修改任何配置文件之前一定要备份原来的文件，这样会让你在修改出错的时候快速恢复到原来状态，换句话说就是改不好，也不能改坏。
```
    # mv /etc/dnsmasq.conf /etc/dnsmasq.conf.20151010
    # vim /etc/dnsmasq.conf
```
4.	编辑配置文件
>由于里面内容太多了，大多也都是注释，所以我全部删除了，重新写了一些比较关键的进去。

    interface=eno16777736,lo  //服务需要监听并提供服务的端口
    domain=centos7pxe.lan  //域名
    dhcp-range=eno16777736,192.168.13.150,192.168.13.200,255.255.255.0,1h
    //dhcp分配IP地址范围
    dhcp-boot=pxelinux.0,pxeserver,192.168.13.132
    //PXE网络引导地址
    dhcp-option=3,192.168.13.132    //网关
    pxe-prompt="Press F8 for menu.",60   //F8进入，默认60秒等待
    pxe-service=x86PC,"PXE Install CentOS 7",pxelinux  
    //标题描述，x86PC,和pxelinux不要更改
    enable-tftp  //启用内建tftp
    tftp-root=/var/lib/tftpboot  //使用默认TFTP目录

配置完成后，保存继续向下配置，后续一起启动。

#### Tftp-Server
安装tftp-server为下载网络引导文件提供传输
1.	安装tftp-server

    #yum –y install tftp-server
	无须配置，稍后启动即可！

#### Syslinux
PXE启动映像文件由syslinux软件包提供，光盘中已提供。
syslinux是一个功能强大的引导加载程序，而且兼容各种介质。更加确切地说：SYSLINUX是一个小型的Linux操作系统，它的目的是简化首次安装Linux的时间，并建立修护或其它特殊用途的启动盘。我们只要安装了syslinux，就会生成一个pxelinux.0，将 pxelinux.0 这个文件复制到 '''tftpboot''' 目录即可：
1.	安装syslinux

    #yum –y install syslinux

2.	拷贝引导文件到tftp目录内

    #cp -r /usr/share/syslinux/* /var/lib/tftpboot

#### Pxe引导文件

其实这个引导文件就如同我们光盘内的isolinux.cfg一样，在最初的状态提供我们一个选择菜单，让我们选择相对应的安装选项，那么问题就简单了，既然一样，那么我们直接就拷贝光盘里的isolinux.cfg过来修改即可。
1.	拷贝文件到tftp指定目录
```
    #mount /dev/cdrom /mnt/cdrom
    #mkdir /var/lib/tftpboot/pxelinux.cfg 
    #cp /mnt/cdrom/isolinux/isolinux.cfg  /var/lib/tftpboot/pxelinux.cfg/default
```
2.	编辑后的default文件如下
```
    default vesamenu.c32
    timeout 600

    menu title @@@PXE BOOT INSTALL MENU @@@
    label local
    menu default
    menu label Boot from ^local drive
    localboot 0xffff

    label Testlinux
    menu label ^Test CentOS 7
    kernel centos7/vmlinuz
    append method=ftp://192.168.13.132/pub initrd=centos7/initrd.img 
    append inst.ks=ftp://192.168.13.132/pub/Base.cfg  initrd=centos7/initrd.img
    //注意kernel 和 initrd部分，指向的centos7目录，这里的目录实际位
    //置/var/lib/tftpboot/centos7目录，安装源的位置在ftp的pub目录
```
>此处的Base.cfg文件为kickstart应答文件，可以参考前面章节`自定义光盘的ks文件`部分。

#### Vmlinux and initrd
1.	拷贝光盘可用的vmlinux和initrd.img到/var/lib/tftpboot/centos7目录，提供可使用的网络启动。
```
    #mkdir /var/lib/tftpboot/centos7
    #cp /mnt/cdrom/images/pxeboot/vmlinux  /var/lib/tftpboot/centos7/
    #cp /mnt/cdrom/images/pxeboot/initrd.img  /var/lib/tftpboot/centos7/
```
#### ftp安装源
基本软件配置已经就绪，那么我们还需一个可以在网络上访问的安装源，你可以采用HTTP，FTP，NFS等协议来提供网络安装源，本文采用VSFTP来建立安装源。
1.	安装配置VSftpd软件
```
    # yum install vsftpd
    # cp -r /mnt/cdrom/* /var/ftp/pub/
    #systemctl restart vsftpd.service
    #systemctl enable vsftpd.service 
```
#### 防火墙
既然是要通过网络提供安装，那么这和防火墙绝对脱不了干系，所以我们要配置防火墙，然后才能正常提供安装。

1.	关闭防火墙

    #systemctl  stop firewalld
    #systemctl  disable firewalld

2.	添加防火墙规则
```
    # firewall-cmd --add-service=ftp –permanent
    # firewall-cmd --add-service=dhcp –permanent
    # firewall-cmd --add-port=69/udp --permanent 
    # firewall-cmd --add-service=dns –permanent   //如果有提供DNS功能的话
    # firewall-cmd --reload
```
#### 启动所有服务
1.	启动所需要的服务
```
    # systemctl start dnsmasq
    # systemctl status dnsmasq
    # systemctl start vsftpd
    # systemctl status vsftpd
    # systemctl enable dnsmasq
    # systemctl enable vsftpd
```
2. 启动tftp-server：
```
    编辑#vim /etc/xinetd.d/tftp配置文件，将disable = yes替换成disable = no
    #systemctl  restart xinetd.service
```
#### Install测试

测试需要将PXE server 和裸机放置在同一网络，并且裸机设置为网络引导启动。
1.	引导成功如下图
![png](./images/pxe/pxe-3.png)
如果该步无法出现，请监控/var/log/messages日志，查看相关错误
2.	引导成功之后选择要安装的系统
![png](./images/pxe/pxe-4.png)
3.	剩下的就是等待安装完成了

## 总结
其实这个PXE使用还是很多的，但是在生产中我会非常小心，我也不会在我的生产中去建立这个东西，因为一旦有疏忽可能会威胁整个生产区域，所以我一般都是将他封装在某个单台主机或者笔记本内，机器安装系统的时候，这些被上架的新主机和我的笔记本首先接到同一交换机上，这个交换机是单独的并且是灵活的，用到哪就拿到哪。然后在开始PXE批量安装。
