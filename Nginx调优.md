Nginx调优

1. 隐藏 Nginx 版本号
```
  为什么要隐藏 Nginx 版本号：一般来说，软件的漏洞都与版本有关，隐藏版本号是为了防止恶意用户利用软件漏洞进行攻击

http {
	...
 	server_tokens   off;     # 隐藏版本号
    ...
}
```
2.隐藏 Nginx 版本号和软件名
```
  为什么要隐藏 Nginx 版本号和软件名：一般来说，软件的漏洞都与版本有关，隐藏版本号是为了防止恶意用户利用软件漏洞进行攻击，而软件名可以进行修改，否则黑客知道是 Nginx 服务器更容易进行攻击，需要注意的是，隐藏 Nginx 软件名需要重新编译安装 Nginx ，如果没有该方面需求尽量不要做

  1) 修改：/usr/local/src/nginx-1.6.3/src/core/nginx.h
#define NGINX_VERSION      "8.8.8.8"                  # 修改为想要显示的版本号
#define NGINX_VER          "Google/" NGINX_VERSION    # 修改为想要显示的软件名
#define NGINX_VAR          "Google"                   # 修改为想要显示的软件名
 2) 修改：/usr/local/src/nginx-1.6.3/src/http/ngx_http_header_filter_module.c
static char ngx_http_server_string[] = "Server: Google" CRLF;  #修改为想要显示的软件名
static char ngx_http_server_full_string[] = "Server: " NGINX_VER CRLF;
 3) 修改：/usr/local/src/nginx-1.6.3/src/http/ngx_http_special_response.c
static u_char ngx_http_error_full_tail[] =
"<hr><center>" NGINX_VER "(www.google.com)</center>" CRLF    # 此行定义对外展示的内容
"</body>" CRLF
"</html>" CRLF
;
 
static u_char ngx_http_error_tail[] =
"<hr><center>Google</center>" CRLF       # 此行定义对外展示的软件名
"</body>" CRLF
"</html>" CRLF
;
 4) 重新编译 Nginx

# cd /usr/local/src/nginx-1.6.3
# ./configure --user=nginx --group=nginx --prefix=/usr/local/nginx --with-http_stub_status_module --with-http_ssl_module
make && make install
/usr/local/nginx/sbin/nginx
```
3.更改 Nginx 服务的默认用户
```
  为什么要更改 Nginx 服务的默认用户：就像更改 ssh 的默认 22 端口一样，增加安全性，Nginx 服务的默认用户是 nobody ，我们更改为 nginx
1) 添加 nginx 用户

# useradd -s /sbin/nologin -M nginx

2) 更改 Nginx 配置文件
user nginx nginx;       # 指定Nginx服务的用户和用户组
```

4.优化 Nginx worker 进程数
```
Nginx 有 Master 和 worker 两种进程，Master 进程用于管理 worker 进程，worker 进程用于 Nginx 服务

worker 进程数应该设置为等于 CPU 的核数，高流量并发场合也可以考虑将进程数提高至 CPU 核数 * 2

# grep -c processor /proc/cpuinfo         # 查看CPU核数
2

# vim /usr/local/nginx/conf/nginx.conf    # 设置worker进程数
worker_processes  2;
```

5.绑定 Nginx 进程到不同的 CPU 上
```
   为什么要绑定 Nginx 进程到不同的 CPU 上 ：默认情况下，Nginx 的多个进程有可能跑在某一个 CPU 或 CPU 的某一核上，导致 Nginx 进程使用硬件的资源不均，因此绑定 Nginx 进程到不同的 CPU 上是为了充分利用硬件的多 CPU 多核资源的目的。

# grep -c processor /proc/cpuinfo    # 查看CPU核数
2
worker_processes  2;         # 2核CPU的配置
worker_cpu_affinity 01 10;
 
worker_processes  4;         # 4核CPU的配置
worker_cpu_affinity 0001 0010 0100 1000;   
 
worker_processes  8;         # 8核CPU的配置
worker_cpu_affinity 00000001 00000010 00000100 00001000 00010000 00100000 01000000 1000000;
```

6.优化 Nginx 处理事件模型
```
  Nginx 的连接处理机制在不同的操作系统会采用不同的 I/O 模型，要根据不同的系统选择不同的事件处理模型，可供选择的事件处理模型有：kqueue 、rtsig 、epoll 、/dev/poll 、select 、poll ，其中 select 和 epoll 都是标准的工作模型，kqueue 和 epoll 是高效的工作模型，不同的是 epoll 用在 Linux 平台上，而 kqueue 用在 BSD 系统中。

(1) 在 Linux 下，Nginx 使用 epoll 的 I/O 多路复用模型
(2) 在 Freebsd 下，Nginx 使用 kqueue 的 I/O 多路复用模型
(3) 在 Solaris 下，Nginx 使用 /dev/poll 方式的 I/O 多路复用模型
(4) 在 Windows 下，Nginx 使用 icop 的 I/O 多路复用模型

# cat /usr/local/nginx/conf/nginx.conf
......
events {
    use epoll;
}
......
```
 7.优化 Nginx 单个进程允许的最大连接数
```
 (1) 控制 Nginx 单个进程允许的最大连接数的参数为 worker_connections ，这个参数要根据服务器性能和内存使用量来调整
 (2) 进程的最大连接数受 Linux 系统进程的最大打开文件数限制，只有执行了 "ulimit -HSn 65535" 之后，worker_connections 才能生效
 (3) 连接数包括代理服务器的连接、客户端的连接等，Nginx 总并发连接数 = worker 数量 * worker_connections, 总数保持在3w左右

# cat /usr/local/nginx/conf/nginx.conf
worker_processes  2;
worker_cpu_affinity 01 10;
user nginx nginx;
events {
    use epoll;
    worker_connections  15000; #单个进程允许的最大连接数
}
......
```

8.优化 Nginx worker 进程最大打开文件数
```
# cat /usr/local/nginx/conf/nginx.conf
......
worker_rlimit_nofile 65535;    # worker 进程最大打开文件数，可设置为优化后的 ulimit -HSn 的结果
......
```

9.优化服务器域名的散列表大小
```
    如下，如果在 server_name 中配置了一个很长的域名，那么重载 Nginx 时会报错，因此需要使用 server_names_hash_max_size 来解决域名过长的问题，该参数的作用是设置存放域名的最大散列表的存储的大小，根据 CPU 的一级缓存大小来设置。

server {
        listen       80;
        server_name  www.abcdefghijklmnopqrst.com;        # 配置一个很长的域名
        .......
}

解决方法：
# cat /usr/local/nginx/conf/nginx.conf
......
http {
    include       mime.types;
    server_names_hash_bucket_size  512;     # 配置在 http 区块，默认是 512kb ，一般设置为 cpu 一级缓存的 4-5 倍，一级缓存大小可以用 lscpu 命令查看
    default_type  application/octet-stream;
    sendfile        on;
    keepalive_timeout  65;
    server_tokens off;
    include vhosts/*.conf;
}
```

 10.开启高效文件传输模式
```
 (1) sendfile 参数用于开启文件的高效传输模式，该参数实际上是激活了 sendfile() 功能，sendfile() 是作用于两个文件描述符之间的数据拷贝函数，这个拷贝操作是在内核之中的，被称为 "零拷贝" ，sendfile() 比 read 和 write 函数要高效得多，因为 read 和 write 函数要把数据拷贝到应用层再进行操作

 (2) tcp_nopush 参数用于激活 Linux 上的 TCP_CORK socket 选项，此选项仅仅当开启 sendfile 时才生效，tcp_nopush 参数可以允许把 http response header 和文件的开始部分放在一个文件里发布，以减少网络报文段的数量
# cat /usr/local/nginx/conf/nginx.conf
......
http {
    include       mime.types;
    server_names_hash_bucket_size  512;    
    default_type  application/octet-stream;
    sendfile      on;    # 开启文件的高效传输模式
    tcp_nopush    on;    # 激活 TCP_CORK socket 选择
    tcp_nodelay on;  #数据在传输的过程中不进缓存
    keepalive_timeout  65;
    server_tokens off;
    include vhosts/*.conf;
}
```

11.优化 Nginx 连接超时时间
```
1. 什么是连接超时
(1) 举个例子，某饭店请了服务员招待顾客，但是现在饭店不景气，因此要解雇掉一些服务员，这里的服务员就相当于 Nginx 服务建立的连接
(2) 当服务器建立的连接没有接收处理请求时，可以在指定的时间内让它超时自动退出

2. 连接超时的作用
(1) 将无用的连接设置为尽快超时，可以保护服务器的系统资源（CPU、内存、磁盘）
(2) 当连接很多时，及时断掉那些建立好的但又长时间不做事的连接，以减少其占用的服务器资源
(3) 如果黑客攻击，会不断地和服务器建立连接，因此设置连接超时以防止大量消耗服务器的资源
(4) 如果用户请求了动态服务，则 Nginx 就会建立连接，请求 FastCGI 服务以及后端 MySQL 服务，设置连接超时，使得在用户容忍的时间内返回数据

3. 连接超时存在的问题
(1) 服务器建立新连接是要消耗资源的，因此，连接超时时间不宜设置得太短，否则会造成并发很大，导致服务器瞬间无法响应用户的请求
(2) 有些 PHP 站点会希望设置成短连接，因为 PHP 程序建立连接消耗的资源和时间相对要少些
(3) 有些 Java 站点会希望设置成长连接，因为 Java 程序建立连接消耗的资源和时间要多一些，这时由语言的运行机制决定的

4. 设置连接超时
(1) keepalive_timeout ：该参数用于设置客户端连接保持会话的超时时间，超过这个时间服务器会关闭该连接
(2) client_header_timeout ：该参数用于设置读取客户端请求头数据的超时时间，如果超时客户端还没有发送完整的 header 数据，服务器将返回 "Request time out (408)" 错误
(3) client_body_timeout ：该参数用于设置读取客户端请求主体数据的超时时间，如果超时客户端还没有发送完整的主体数据，服务器将返回 "Request time out (408)" 错误
(4) send_timeout ：用于指定响应客户端的超时时间，如果超过这个时间，客户端没有任何活动，Nginx 将会关闭连接
(5) tcp_nodelay ：默认情况下当数据发送时，内核并不会马上发送，可能会等待更多的字节组成一个数据包，这样可以提高 I/O 性能，但是，在每次只发送很少字节的业务场景中，使用 tcp_nodelay 功能，等待时间会比较长

# cat /usr/local/nginx/conf/nginx.conf
......
http {
    include       mime.types;
    server_names_hash_bucket_size  512;    
    default_type  application/octet-stream;
    sendfile        on;
    keepalive_timeout  65;
    tcp_nodelay on;
    client_header_timeout 15;
    client_body_timeout 15;
    send_timeout 25;
    include vhosts/*.conf;
}
```

12.限制上传文件的大小
```
   client_max_body_size 用于设置最大的允许客户端请求主体的大小，在请求首部中有 "Content-Length" ，如果超过了此配置项，客户端会收到 413 错误，即请求的条目过大

http {
......
    client_max_body_size 8m;    # 设置客户端最大的请求主体大小为8M}
......
}
```

13.FastCGI 相关参数调优 
```
    当 LNMP 组合工作时，首先是用户通过浏览器输入域名请求 Nginx Web 服务，如果请求的是静态资源，则由 Nginx 解析返回给用户；如果是动态请求（如 PHP），那么 Nginx 就会把它通过 FastCGI 接口发送给 PHP 引擎服务（即 php-fpm）进行解析，如果这个动态请求要读取数据库数据，那么 PHP 就会继续向后请求 MySQL 数据库，以读取需要的数据，并最终通过 Nginx 服务把获取的数据返回给用户，这就是 LNMP 环境的基本请求流程。

 FastCGI 介绍：CGI 通用网关接口，是 HTTP 服务器与其他机器上的程序服务通信交流的一种工具，CGI 接口的性能较差，每次 HTTP 服务器遇到动态程序时都需要重新启动解析器来执行解析，之后结果才会被返回 HTTP 服务器，因此就有了 FastCGI ，FastCGI 是一个在 HTTP 服务器和动态脚本语言间通信的接口，主要是把动态语言和 HTTP 服务器分离开来，使得 HTTP 服务器专一地处理静态请求，提高整体性能，在 Linux 下，FastCGI 接口即为 socket ，这个 socket 可以是文件 socket 也可以是 IP socket

# cat /usr/local/nginx/conf/nginx.conf
worker_processes  1;
events {
    worker_connections  1024;
}
http {
    include       mime.types;
    default_type  application/octet-stream;
    sendfile        on;
    keepalive_timeout  65;
    fastcgi_connect_timeout  240;    # Nginx服务器和后端FastCGI服务器连接的超时时间
    fastcgi_send_timeout     240;    # Nginx允许FastCGI服务器返回数据的超时时间，即在规定时间内后端服务器必须传完所有的数据，否则Nginx将断开这个连接
    fastcgi_read_timeout     240;    # Nginx从FastCGI服务器读取响应信息的超时时间，表示连接建立成功后，Nginx等待后端服务器的响应时间
    fastcgi_buffer_size      64k;    # Nginx FastCGI 的缓冲区大小，用来读取从FastCGI服务器端收到的第一部分响应信息的缓冲区大小
    fastcgi_buffers        4 64k;    # 设定用来读取从FastCGI服务器端收到的响应信息的缓冲区大小和缓冲区数量
    fastcgi_busy_buffers_size    128k;    # 用于设置系统很忙时可以使用的 proxy_buffers 大小
    fastcgi_temp_file_write_size 128k;    # FastCGI 临时文件的大小
#   fastcti_temp_path            /data/ngx_fcgi_tmp;    # FastCGI 临时文件的存放路径
    fastcgi_cache_path           /data/ngx_fcgi_cache  levels=2:2  keys_zone=ngx_fcgi_cache:512m  inactive=1d  max_size=40g;    # 缓存目录
     
    server {
        listen       80;
        server_name  www.abc.com;
        location / {
            root   html/www;
            index  index.html index.htm;
        }
        location ~ .*\.(php|php5)?$ {
            root            html/www;
            fastcgi_pass    127.0.0.1:9000;
            fastcgi_index   index.php;
            include         fastcgi.conf;
            fastcgi_cache   ngx_fcgi_cache;            # 缓存FastCGI生成的内容，比如PHP生成的动态内容
            fastcgi_cache_valid      200  302  1h;     # 指定http状态码的缓存时间，这里表示将200和302缓存1小时
            fastcgi_cache_valid      301  1d;          # 指定http状态码的缓存时间，这里表示将301缓存1天
            fastcgi_cache_valid      any  1m;          # 指定http状态码的缓存时间，这里表示将其他状态码缓存1分钟
            fastcgi_cache_min_uses   1;                # 设置请求几次之后响应被缓存，1表示一次即被缓存
            fastcgi_cache_use_stale  error  timeout  invalid_header  http_500;    # 定义在哪些情况下使用过期缓存
            fastcgi_cache_key        http://$host$request_uri;                    # 定义 fastcgi_cache 的 key
        }
    }
}
```

 14.配置 Nginx gzip 压缩
```
   Nginx gzip 压缩模块提供了压缩文件内容的功能，用户请求的内容在发送到客户端之前，Nginx 服务器会根据一些具体的策略实施压缩，以节约网站出口带宽，同时加快数据传输效率，来提升用户访问体验，需要压缩的对象有 html 、js 、css 、xml 、shtml ，图片和视频尽量不要压缩，因为这些文件大多都是已经压缩过的，如果再压缩可能反而变大，另外，压缩的对象必须大于 1KB，由于压缩算法的特殊原因，极小的文件压缩后可能反而变大

# cat /usr/local/nginx/conf/nginx.conf
......
http {
    gzip  on;                    # 开启压缩功能
    gzip_min_length  1k;         # 允许压缩的对象的最小字节
    gzip_buffers  4 32k;         # 压缩缓冲区大小，表示申请4个单位为16k的内存作为压缩结果的缓存
    gzip_http_version  1.1;      # 压缩版本，用于设置识别HTTP协议版本
    gzip_comp_level  9;          # 压缩级别，1级压缩比最小但处理速度最快，9级压缩比最高但处理速度最慢
    gzip_types  text/css text/xml application/javascript;    # 允许压缩的媒体类型
    gzip_vary  on;               # 该选项可以让前端的缓存服务器缓存经过gzip压缩的页面，例如用代理服务器缓存经过Nginx压缩的数据
}
```

16.优化 Nginx access 日志
```
1. 配置日志切割

# vim /usr/local/nginx/conf/cut_nginx_log.sh
#!/bin/bash
savepath_log='/usr/local/clogs'
nglogs='/usr/local/nginx/logs'
mkdir -p $savepath_log/$(date +%Y)/$(date +%m)
mv $nglogs/access.log $savepath_log/$(date +%Y)/$(date +%m)/access.$(date +%Y%m%d).log
mv $nglogs/error.log $savepath_log/$(date +%Y)/$(date +%m)/error.$(date +%Y%m%d).log
kill -USR1 `cat /usr/local/nginx/logs/nginx.pid`
[root@localhost ~]# crontab -e    # 每天凌晨0点执行脚本
0 0 * * * /bin/sh /usr/local/nginx/conf/cut_nginx_log.sh > /dev/null 2>&1

 2. 不记录不需要的访问日志

location ~ .*\.(js|jpg|JPG|jpeg|JPEG|css|bmp|gif|GIF)$ {
    access_log  off;
}
 3. 设置访问日志的权限

chown -R root.root /usr/local/nginx/logs
chmod -R 700 /usr/local/nginx/logs
```

 17.优化 Nginx 站点目录
```
 1. 禁止解析指定目录下的指定程序
location ~ ^/data/.*\.(php|php5|sh|pl|py)$ {     # 根据实际来禁止哪些目录下的程序，且该配置必须写在 Nginx 解析 PHP 的配置前面
    deny all;
}
 2. 禁止访问指定目录
location ~ ^/data/.*\.(php|php5|sh|pl|py)$ {     # 根据实际来禁止哪些目录下的程序，且该配置必须写在 Nginx 解析 PHP 的配置前面
    deny all;
}
 3. 限制哪些 IP 不能访问网站
location ~ ^/wordpress {     # 相对目录，表示只允许 192.168.1.1 访问网站根目录下的 wordpress 目录
    allow 192.168.1.1/24;
    deny all;
}
```

 18.配置 Nginx 防盗链
```
  什么是防盗链：简单地说，就是某些不法网站未经许可，通过在其自身网站程序里非法调用其他网站的资源，然后在自己的网站上显示这些调用的资源，使得被盗链的那一端消耗带宽资源  (1) 根据 HTTP referer 实现防盗链：referer 是 HTTP的一个首部字段，用于指明用户请求的 URL 是从哪个页面通过链接跳转过来的

(2) 根据 cookie 实现防盗链：cookie 是服务器贴在客户端身上的 "标签" ，服务器用它来识别客户端

根据 referer 配置防盗链：

#第一种,匹配后缀
location ~ .*\.(gif|jpg|jpeg|png|bm|swf|flv|rar|zip|gz|bz2)$ {    # 指定需要使用防盗链的媒体资源
    access_log  off;                                              # 不记录防盗链的日志
    expires  15d;                                                 # 设置缓存时间
    valid_referers  none  blocked  *.test.com  *.abc.com;         # 表示这些地址可以访问上面的媒体资源
    if ($invalid_referer) {                                       # 如果地址不如上面指定的地址就返回403
        return 403
    }
}
 
#第二种,绑定目录
location /images {  
    root /web/www/img;
    vaild_referers nono blocked *.spdir.com *.spdir.top;
    if ($invalid_referer) {
        return 403;
    }
}
```
 19.配置 Nginx 错误页面优雅显示
```
# cat /usr/local/nginx/conf/nginx.conf
......
http {
    location / {
        root   html/www;
        index  index.html index.htm;           
        error_page 400 401 402 403 404 405 408 410 412 413 414 415 500 501 502 503 506 = http://www.xxxx.com/error.html;
        # 将这些状态码的页面链接到 http://www.xxxx.com/error.html ，也可以单独指定某个状态码的页面，如 error_page 404 /404.html
    }
}
```

 20.优化 Nginx 文件权限
```
   为了保证网站不受木马入侵，所有文件的用户和组都应该为 root ，所有目录的权限是 755 ，所有文件的权限是 644

# chown -R root.root /usr/local/nginx/....   # 根据实际来调整
# chmod 755 /usr/local/nginx/....
# chmod 644 /usr/local/nginx/....
```

21.Nginx 防爬虫优化
```
 我们可以根据客户端的 user-agents 首部字段来阻止指定的爬虫爬取我们的网站\

if ($http_user_agent ~* "qihoobot|Baiduspider|Googlebot|Googlebot-Mobile|Googlebot-Image|Mediapartners-Google|Adsbot-Google|Yahoo! Slurp China|YoudaoBot|Sosospider|Sogou spider|Sogou web spider|MSNBot") {
    return 403;
}
```
 22. 集群代理优化
```
upstream bbs_com_pool {  #定义服务器池
    ip_hash;    #会话保持(当服务器集群中没有会话池时，且代理的是动态数据就必须写ip_hash,反之什么也不用写)
    #fair   #智能分配(第三方，需要下载upstream_fair模块)根据后端服务器的响应时间来调度
    #url_hash   #更具URL的结果来分配请求(每个url定向到同一个服务器,提高后端缓存服务器的效率，本身不支持，需要安装nginx_hash)
     
    #当算法为ip_hash不能有 weight backup
    server 192.168.10.1:80;     #默认weight为1
    server 192.168.10.3:80 weight=5;    #weight表示权重
    server 192.168.10.4:80 down;    #down:不参与本次轮询
    server 192.168.10.5:80 down backup; #backup:当任何一台主机出现故障，将进行切换替换
    server 192.168.10.6:80 max_fails=3 fail_timeout=20s; #max_fails最大失败请求次数(默认为1)，fail_timeout失败超时时间
    server 192.168.10.7:8080;
     
}

stream{
    upstream cluster {
        # hash $remote_addr consistent; //保持 session 不变,四层开启
        server 192.168.1.2:80 max_fails=3 fail_timeout=30s;
        server 192.168.1.3:80 max_fails=3 fail_timeout=30s;
    }
    server {
        listen 80;
        proxy_pass cluster;
        proxy_connect_timeout 1s;
        proxy_timeout 3s;
    }

    location {
        proxy_next_upstream http_500 http_502 http_503 error timeout invalid_header; 当发生其中任何一种错误，将转交给下一个服务器
        proxy_redirect off;
        proxy_set_header Host &$host;   #设置后端服务器的真实地址
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded_For &proxy_add_x_forwarded_for;
        client_body_buffer_size 128k;       #缓冲区大小，本地保存大小
        proxy_connect_timeout 90;   #发起握手等待的响应时间
        proxy_read_timeout 90;  #建立连接后等待后端服务器响应时间(其实是后端等候处理的时间)
        proxy_send_timeout 90;  #给定时间内后端服务器必须响应，否则断开
        proxy_buffer_size 4k;   #proxy缓冲区大小
        proxy_buffers 4 32k;    #缓冲区个数和大小
        proxy_busy_buffers_size 64k;    #系统繁忙时buffer的临时大小，官方要求proxy_buffer_size*2
        proxy_temp_file_write_size 64k; #proxy临时文件的大小
    }
}
```
