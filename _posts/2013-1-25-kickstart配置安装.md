---
layout : post
category : 学习
tags : [kickstart配置安装]
title : kickstart配置安装
---
[思维导图文件下载](#) 
#kickstart配置安装   
以redhat5.4 X64为例  
###1.首先挂载iso，然后把iso复制到指定文件下
    mount /dev/cdrom /media/cdrom
    mkdir /kickstart          #建立新分区的挂载目录
    cp -r ./* /kickstart          #拷贝/mnt/cdrom下所有文件到/kickstart
    echo "/kickstart  *(rw,sync)" > /etc/exports     #设置nfs共享
    
    service portmap start          #启动portmap服务
    service nfs start          #启动nfs服务
    chkconfig portmap on
    chkconfig nfs on
    showmount –e          #查看nfs共享
    
    yum -y install tftp-server
    yum -y install dhcp dhcp-devel
    yum -y install system-config-kickstart.noarch
    
    vim /etc/xinetd.d/tftp          #tfpt配置文件
    disable = no               #将此处的yes改为no即可
    service xinetd start          #启动xinetd服务
    chkconfig xinetd on

###2.kickstart配置
    cp /mnt/cdrom/isolinux/vmlinuz /tftpboot/     #拷贝光盘中vmlinuz到/tftpboot下
    cp /mnt/cdrom/isolinux/initrd.img /tftpboot/     #拷贝光盘中initrd.img到/tftpboot下
    cp /usr/lib/syslinux/pxelinux.0 /tftpboot/     #拷贝本地pxelinux.0到/tftpboot下
    
    mkdir /tftpboot/pxelinux.cfg
cp /mnt/cdrom/isolinux/isolinux.cfg /tftpboot/pxelinux.cfg/default    #拷贝光盘中isolinux.cfg到/tftpboot/pxelinux.cfg目录下的default文件  

vim /tftpboot/pxelinux.cfg/default

    default 1
    prompt 1
    timeout 600
    display boot.msg
    F1 boot.msg
    F2 options.msg
    F3 general.msg
    F4 param.msg
    F5 rescue.msg
    label 1
      localboot 1
    label 2
      kernel vmlinuz_64
      append initrd=initrd_rhel5_64.img ks=nfs:192.168.110.15:/kickstart/ks.cfg ksdevice=eth0    #指定IOS文件地址及安装配置文件

cp /usr/share/doc/dhcp-3.0.5/dhcpd.conf.sample /etc/dhcpd.conf       #拷贝dhcp配置文件模板到/etc下  
vim /etc/dhcpd.conf  

    subnet 192.168.110.0 netmask 255.255.255.0 {
            option routers                  192.168.110.1;
            option subnet-mask              255.255.255.0;
    
            option nis-domain               "domain.org";
            option domain-name              "domain.org";
            option domain-name-servers      192.168.1.10;
    
            option time-offset              -18000; # Eastern Standard Time
            host ns01 {
                    hardware ethernet 00:0C:29:af:26:3f;
                    fixed-address 192.168.110.16;
            }
    }

3.生成ks.cfg文件  
system-config-kickstart          #生成ks.cfg文件,生成后指定存放位置   
在/tftpboot/pxelinux.cfg/default 已经指定了ks.cfg的存放位置  
vim  /kickstart/ks.cfg  

    #platform=x86, AMD64, or Intel EM64T
    # System authorization information
    auth  --useshadow  --enablemd5
    # System bootloader configuration
    bootloader --location=mbr
    # Clear the Master Boot Record
    zerombr
    # Partition clearing information
    clearpart --all --initlabel
    key --skip
    # Use text mode install
    text
    # Firewall configuration
    firewall --disabled
    # Run the Setup Agent on first boot
    firstboot --disable
    # System keyboard
    keyboard us
    # System language
    lang en_US.UTF-8
    # Installation logging level
    logging --level=info
    # Use NFS installation media
    nfs --server=192.168.200.201 --dir=/opt/software/Kickstart/RHEL5U4_64
    # Network information
    #network --bootproto=dhcp --device=eth0 --onboot=on --hostname
    network --bootproto=dhcp --device=eth0 --onboot=on
    # Reboot after installation
    reboot
    #Root password
    rootpw --iscrypted $1$hDrpHFPO$nvQciiqJet3zaZncrOIFx.  ###root密码设置为“rhelroot”
    
    # SELinux configuration
    selinux --disabled
    # System timezone
    timezone --isUtc Asia/Shanghai
    # Install OS instead of upgrade
    install
    # X Window System configuration information
    xconfig  --defaultdesktop=GNOME --depth=32 --resolution=800x600 --startxonboot
    # Disk partitioning information
    part /boot --bytes-per-inode=4096 --fstype="ext3" --size=100
    part swap --bytes-per-inode=4096 --fstype="swap" --size=4096
    part / --asprimary --bytes-per-inode=4096 --fstype="ext3" --grow --size=1
    
    %packages
    @base
    @gnome-desktop
    @development-libs
    @admin-tools
    @sound-and-video
    @chinese-support
    @gnome-software-development
    @development-tools
    @x-software-development
    @office
    @legacy-software-support
    @printing
    @base-x
    @text-internet
    @dialup
    @graphics
    @graphical-internet
    @editors
    @java
    @games
    
    %post
    ###系统安装后执行的shell脚本#######
###3.设置无人值守安装界面
vim /tftpboot/boot.msg   #如果无此文件，可以从/mnt/isolinux 中复制过来


    ^L
    ^Xsplash.lss
    
    -  To install or upgrade in graphical mode, press the ^O01<ENTER>^O07 key.
    
    -  To install or upgrade in text mode, type: ^O01linux text <ENTER>^O07.
    
    -  Use the function keys listed below for more information.
    
    
             1. Local disk boot(Default)    #序列和/tftpboot/pxelinux.cfg/default要一一对应
             2. Linux RHEL5U4_32 install
             3. Linux RHEL5U4_64 install (only install nagios client and ldap auth)
             4. Linux RHEL5U4_64 install (install nagios client and LAP)
             5. Linux RHEL5U4_64 install (install nagios client and MYSQL)
             6. Linux RHEL5U4_64 install (install nagios client and LNP)
             7. Linux RHEL5U4_64 install (install nagios client and LNMP)
             8. Linux RHEL5U4_64 install (install nagios client and LAMP)

    
