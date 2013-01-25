---
layout : post
category : 学习
tags : [varnish配置 web加速]
title : varnish配置 web加速
---
[思维导图文件下载](#) 
##一、 Varnish介绍  
Varnish与 一般服务器软件类似，分为master（management）进程和child（worker，主要做cache的工作）进程。master进程读入命 令，进行一些初始化，然后fork并监控child进程。child进程分配若干线程进行工作，主要包括一些管理线程和很多woker线程。针对文件缓存 部分，master读入存储配置（-s file[,path[,size[,granularity]]] ），调用合适的存储类型，然后创建/读入相应大小的缓存大文件（根据其man文档，为避免文件出现存储分片[2]影响读写性能，作者建议用dd(1)命令 预先创建大文件）。接着，master初始化管理该存储空间的结构体。这些变量都是全局变量，在fork以后会被child进程所继承（包括文件描述 符）。  
在child进程主线程初始化过程中，将前面打开的存储大文件整个mmap到内存中（如果超出系统的虚拟内存，mmap失败，进程会减少原来的配置mmap大小，然后继续mmap），此时创建并初始化空闲存储结构体，挂到存储管理结构体，以待分配。  

接着，真正的工作开始，Varnish的某个负责接受新HTTP连接的线程开始等待用户，如果有新的HTTP连接过来， 它总负责接收，然后叫醒某个等待中的线程，并把具体的处理过程交给它。Worker线程读入HTTP请求的URI，查找已有的object，如果命中则直 接返回并回复用户。如果没有命中，则需要将所请求的内容，从后端服务器中取过来，存到缓存中，然后再回复。  

分配缓存的过程是这样的：它根据所读到object的大小，创建相应大小的缓存文件。为了读写方便，程序会把每个object的大小变为最接近其 大小的内存页面倍数。然后从现有的空闲存储结构体中查找，找到最合适的大小的空闲存储块，分配给它。如果空闲块没有用完，就把多余的内存另外组成一个空闲 存储块，挂到管理结构体上。如果缓存已满，就根据LRU[4]机制，把最旧的object释放掉。  

释放缓存的过程是这样的：有一个超时线程，检测缓存中所有object的生存期，如果超初设定的TTL（Time To Live）没有被访问，就删除之，并且释放相应的结构体及存储内存。注意释放时会检查该存储内存块前面或后面的空闲内存块，如果前面或后面的空闲内存和该 释放内存是连续的，就将它们合并成更大一块内存。整个文件缓存的管理，没有考虑文件与内存的关系，实际上是将所有的object都考虑是在内存中，如果系 统内存不足，系统会自动将其换到swap空间，而不需要varnish程序去控制。  
##二、varnish配置安装  
####1.安装配置  
    yum  -y install automake autoconf libtool ncurses-devel libxslt groff pcre-devel pkgconfig
    export PKG_CONFIG_PATH=/usr/local/lib/pkgconfig
    tar -xf varnish-3.0.2.tar.gz  && cd varnish-3.0.2 && ./configure --prefix=/usr/local/varnish && make && make install
    cp redhat/varnish.initrc /etc/init.d/varnish
    cp redhat/varnish.sysconfig /etc/sysconfig/varnish
    cp redhat/varnish_reload_vcl /usr/local/varnish/bin/
    
    vim /etc/init.d/varnish
    exec="/usr/local/varnish/sbin/varnishd"
    reload_exec="/usr/local/varnish/bin/varnish_reload_vcl"
 #修改这两行，与varnish安装路径有关  
 
    vim /etc/sysconfig/varnish
    RELOAD_VCL=1
    VARNISH_VCL_CONF=/usr/local/varnish/etc/varnish/default.vcl
    VARNISH_LISTEN_PORT=80
    VARNISH_STORAGE_SIZE=128M    #缓存数据的大小
 #修改如上4行，根据具体情况而定  
vim /usr/local/varnish/bin/varnish_reload_vcl  
注释掉关于认证文件的部分：  

    #elif [ -z "$VARNISH_SECRET_FILE" ]; then
    #       echo "Warning: VARNISH_SECRET_FILE is not set"
    #       secret=""
    #elif [ ! -s "$VARNISH_SECRET_FILE" ]; then
    #       echo "Error: varnish secret file $VARNISH_SECRET_FILE is unreadable or empty"
    #       exit 2
    #else
    #       secret="-S $VARNISH_SECRET_FILE"
    #*******找到定义varnishadm的行，指定路径：***********#
    # Done parsing, set up command
    VARNISHADM="/usr/local/varnish/bin/varnishadm $secret -T $VARNISH_ADMIN_LISTEN_ADDRESS:$VARNISH_ADMIN_LISTEN_PORT"
####2、增加用户
建立一个varnish的日志目录，如果需要日志输出时使用  

     useradd -s /bin/false -M varnish
     mkdir /var/log/varnish/
     chown -R varnish.varnish /var/log/varnish/
####3、varnish主要配置文件配置  
下面是一个基本的配置文件。test.vcl包含了大部分的基本配置。把注释去掉，再加上自定义的一些指令即可完成  
路径：/usr/local/varnish/etc/varnish/  
vim /usr/local/varnish/etc/varnish/test.vcl   

    backend  img {
    .host = "192.168.110.20";
    .port = "80";
    }
    
    backend  www {
    .host = "192.168.110.20";
    .port = "80";
    }
    
    backend  netseek {
    .host = "192.168.110.20";
    .port = "80";
    }
    
    #acl
    acl purge {
      "localhost";
      "127.0.0.1";
      "192.168.169.0"/24;
    }
    
    sub vcl_recv {
    if (req.request == "PURGE") {
    if (!client.ip ~ purge)
    {
    error 405 "Not allowed.";
    }
    return(lookup);
    }
    
    if (req.http.host ~ "^192.168.110.20") {
    set req.backend = img;}
    elseif (req.http.host ~ "^(www)|(bbs)|(doc).linuxtone.org") {
    set req.backend = www;}
    elseif (req.http.host ~ ".netseek.com") {
    set req.backend = netseek;}
    else {
    #error 404 "the server is wrong!";
    set req.backend = www;
    }
    
    if (req.request != "GET" && req.request != "HEAD") {
    return(pipe);
    }
    elseif (req.url ~ "\.(php|cgi)($|\?)")
    {
    return(pass);
    }
    return(lookup);
    }
    
    sub vcl_hit {
    if (req.request == "PURGE") {
    set obj.ttl = 0s;
    error 200 "Purged.";
    }
    }

    sub vcl_miss {
    if (req.request == "PURGE") {
    error 404 "Not in cache.";
    }
    }
备注：上面这段主要是抓取缓存数据的配置文件，根据自己的实际情况配置  
####4、启动varnish
    /usr/local/varnish/sbin/varnishd -f /usr/local/varnish/etc/varnish/test.vcl  -s  malloc,1G  -T 127.0.0.1:3000  -w 1000,51200,10  -a  0.0.0.0:8080
    /usr/local/varnish/bin/varnishncsa -a -w /usr/local/varnish/logs  
    /varnish.log &
备注：如果看到不日志，试下把apache执行用户和varnish设置成一致的。   
 #参数说明    
 
    -n vcache       #临时文件实例名.如果以"/"开头,就必须是一个可用的路径.
    -a IP:80        #服务所在端口.":80"是默认所有网络都建立80端口,":"前面是服务器IP.
    -T :5000       #管理端口.
    -s malloc,0.5G    # malloc仅使用内存进行存储，根据实际设置内存大小
    -s file,/data2/vcache,80g  # file是mmap的文件内存映射机制（一般不用，要不使用Varnish就没有意义了）
    -f /usr/local/varnish/etc/varnish.vcl            #VCL文件路径.
    -P /var/run/varnish.pid       #PID文件地址.
    -w 100,2000,10      #工作进程数.三个参数分别是:<min>,<max>,<timeout>
    -h classic,16383        #hash列表类型,以及长度.默认长度是16383
    -g www -u www          #服务运行用户和用户组配置.
    -p thread_pools=8       #进程connections pools的个数,数量越多,越耗用cpu和内存但是处理并发能力越强 ，官方文档建议一个CPU一个
    -p listen_depth=1024     #TCP队列长度.默认是512. 
##5、varnish测试
     http://192.168.110.20:8080/   #看见网站首页
    /usr/local/varnish/logs/varnish.log  #有日志生成
    /usr/local/varnish/bin/varnishstat    
    #能看到cache-hit




