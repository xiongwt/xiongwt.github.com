---
layout : post
category : 学习
tags : [Rsync+inotify-tools数据同步]
title : Rsync+inotify-tools数据同步
---
[思维导图文件下载](#) 
#Rsync+inotify-tools数据同步
###一、rsync数据同步
在实现rsync数据同步之前需要实现服务器之间互信或者认证，下面分别介绍下这两种方法  
服务器A：192.168.1.2   
服务器B：192.168.1.3  
#####1.ssh互信  
服务器A： 

    ssh-keygen -t rsa           #生成密钥，一直按回车即可
    cd /root/.ssh/              #生成密钥的位置
    echo > authorized_keys     #建一个空的文件，用于存放服务器B的公钥
备注：在/root/.ssh会生成两个文件id_rsa.pub和id_rsa.前一个是公钥，后一个是私钥。  
把服务器A的公钥中的内容拷贝到服务器B的 /root/.ssh/authorized_keys中，并把服务器A的/etc/ssh/sshd_config中的参数PermitRootLogin设置为 yes。最后重启sshd服务。  
测试：在服务器B上： ssh 192.168.1.2  
服务器B操作和A一样。  
数据同步rsync命令（在服务器B上执行）  

    #!/bin/bash 
    /usr/bin/rsync -vzrtogpg --delete  root@192.168.1.2:/usr/local/nginx/Portal/  /usr/local/nginx/Portal >> /var/log/rsync_$(date +%Y%m%d).log
    find /var/log/ -mtime +7  -name "rsync_*" | xargs rm -rf

#####2.rsync自定义密码认证
服务器B同步服务器A的数据  
在服务器A的/etc目录下建立两个文件rsyncd.conf和rsyncd.pass  
Rsync.conf文件内容如下：  

     uid = nobody  ##全局配置开始，指文件传输时模块进程的uid gid = nobody              ##同上gid 
     pid file = /var/run/rsyncd.pid   ##pid位置 
     [rsync]                     ##模块配置开始 
     path = /opt/backup         ##需要备份的目录，必须指定， 
     comment = whole backup area   ##注释 
     read only = no            ##客户端是否只读 
     write only = no           ##是否只能写 
     hosts allow = *           ##允许同步主机 
     hosts deny = 192.168.0.0/24  ##禁止访问的主机 
     list = yes                ##是否允许列出所有模块 
     auth users = slave          ##可以连接该模块的user 
     secrets file = /etc/rsync.pass  ##密码文件在哪，需要自己建立
建立密码文件 /etc/rsync.pass 如下格式，并确保权限为600或400

    slave:helloworld 
数据同步rsync命令（在服务器B上执行）  

        #!/bin/bash 
        /usr/local/bin/rsync -vzrtogpg --delete --progress \ 
        slave@192.168.1.2::rsync /opt/backup --password-file=/root/rsync.pass

###二、Inotify
inotify 是一种强大的，细粒度的异步文件系统事件监控机制。通过inotify可以监控文件系统中的添加、删除、修改等，利用这个内核接口，第三方的软件可以监控文件系统的变化，从而触发rsync的同步操作，我们用inotify-tools来实现这个功能。  
下载： 

    http://cloud.github.com/downloads/rvoicilas/inotify-tools/inotify-tools-3.14.tar.gz
    tar xvf inotify-tools-3.14.tar.gz  
    cd inotify-tools-3.14 
    ./configure 
    make && make install  
生成了两个执行程序 usr/local/bin/inotifywait  /usr/local/bin/inotifywatch，inotifywait用来监控文件系统的更改，inotifywatch用来统计更改文件系统事件的。  
inotifywait的一些参数 

    -m  --monitor     ##始终监控 
    -r  --recursive   ##递归的 
    -q  --quiet       ##打印监控事件 
    -e  --event       ##指出要监控的事件，有：modify,delete,create,attrib等
运行 inotifywait -mrq --timefmt '%d/%m/%y %H:%M' --format '%T %w%e' -e modify,delete,create,attrib /opt/backup 往/opt/backup中添加一个文件，查看有没有输出，如果有，代表一切正常。 

    --timefmt  时间格式 
    --format   变化文件的详细信息  
写一个脚本来实现，当/var/ftp/pub/中文件有变化时，让slave同步   
vi inotify_slave.sh 
 
    #!/bin/bash 
    inotifywait -mrq --timefmt '%d/%m%y %H%M' --format '%T %w%f%e' \
    -e modify,delete,create,attrib /var/ftp/pub  | while read files  
    do 
    ssh 192.168.1.3 '/root/rsync.sh'  ##双机互信已经做好
    done 
运行脚本，在/var/ftp/pub中添加文件测试  

    1 sh inotify_slave.sh &  
    2 cp -R /etc/rc.d/init.d /opt/backup 
查看192.168.1.3中文件是否同步  