#系统说明

###
在ios开发中我们遇到很多问题,ios并不像android那样生成一个apk就可以直接安装,安装一个已经签名的ipa都要走很多流程,苹果的文档也是惨不忍睹, 本系统是为了解决一些常见问题,我之前从事的是uniapp开发如果大家接触过都知道,当你开发完毕后上传ipa审核都需要mac系统非常的繁琐,
不仅仅是mdm系统,后续我会出更多关于ios的工具接口,接口化是为了方便调用,我的未来计划

mdm系统-完善
跨平台签名ipa-使用linux签名
应用分发 ios在开发中给用户手机安装需要走ios的内侧通道 需要先获取udid打专用包非常繁琐 一个链接 直接让这些流程自动化
apple应用管理 在windwos中也可以上传ipa审核


###mdm模块

####说明:
mdm是ios为了企业开发制作的一套移动设备管理系统,可以理解为远程控制多个设备,安装卸载移除数据等等,这套方案适用于
一些企业.比如开发内部app,每个员工都需要装有这个app,公司高管可以直接控制用户手机,或者企业提供手机,企业涉及机密,需要完全的
可以控制,在此我们不讨论隐私权的问题

mdm服务器就是一台简单的云服务器,由我们自己控制

mdm流程:
1. mdm服务器使用苹果的mdm证书和苹果的apns服务器进行链接 本系统已经完成这个步骤 我们只需要上传证书即可自动管理 而且支持多个证书 上传后会返回证书id后续操作需要用到 
![123.png](https://s1.ax1x.com/2022/04/15/LGMRkq.png)
2. 注册手机设备,如何注册? 制作一个描述文件mobileconfig 让用户安装即可 此api填入证书id 会返回一个配置文件下载 链接 
![img.png](https://s1.ax1x.com/2022/04/15/LGQGCV.png)
用户使用safari浏览器访问即可,即可下载描述文件,用户需要手动去设置-通用-描述文件与设备管理安装
![img1.png](https://s1.ax1x.com/2022/04/15/LGl4L4.jpg)
当用户完成安装后设备信息就注册在了mdm服务器里 通过设备管理可以查询到该设备 后面对设备操作都会使用到这个deviceid
![img2.png](https://s1.ax1x.com/2022/04/15/LG1HBQ.png)   
3. 向设备发送命令 命令有很多我们暂时只演示安装app 这里的plist文件可以在其他工具接口制作 会返回一个任务id 任务id可以查看这个任务是否执行成功和获取手机信息的一些返回 在此不在演示
![img4.png](https://s1.ax1x.com/2022/04/15/LG8BLD.png)
用户设备会受到安装确认信息
![img5](https://s1.ax1x.com/2022/04/15/LGtMtJ.png)   
到此整个流程结束 
   
只需要调用几个接口即可完成整个mdm的功能,只需要关系自己的业务逻辑即可

本套系统并不开源,只提供可部署的jar包,如需使用可以自行搭建到自己的服务器
当然 你可能会担心不开源是否会有漏洞? 本套系统并不建议对外开放 只需内部使用即可
不开源会不会有后门?偷取证书? 我只能说 不会 如果担心可以不用 


本套系统仅是为了帮助大家快速开发 请勿用于违法行为 

#搭建
##准备:
### 使用 docker
1. centos7干净的服务器一台 就是刚重装系统的样子 防火墙和安全组必须开放 80 443 10800 10801四个端口 
2. 备案域名 如果你是国外服务器则不需要备案
3. ssl证书 ssl证书必须是ca认证 不支持自签证书 需要nginx格式 也就是pem和key两个证书文件

安装本服务器很简单 只需要一条命令 等待命令完成
```aidl
yum install docker -y && systemctl start docker &&  systemctl enable docker && docker run  -tdi --privileged  -p 10800:10800 -p 10801:10801 --name mdm -d  --restart always docker.io/2524931333/centos7mdm:v1 init
```

命令完成后其实已经启动成功你可以通过 http://你的服务器公网ip/doc.html 访问接口文档

但事实上 ip访问是不允许的 仅做查看服务是否启动使用 请勿使用ip去请求api 我们需要配置域名和https

1. 配置域名如果你已经安装了宝塔 直接在宝塔里面配置转发到 10801端口即可 理论是这样 但实际没测试过

2. 如果你不使用宝塔可以使用下面方案

执行命令
```aidl
yum install -y nginx && systemctl start nginx && systemctl enable nginx
```


使用vim或者ftp进入服务器 修改文件 /etc/nginx/nginx.conf 替换为下面的内容 
注意下面的内容有两处需要修改 替换你的域名
然后将ssl nginx的证书 分别修改为 cert.pem 和 cert.key 放入 /etc/nginx/cert/ 目录下 这个cert目录需要新建 
并且给予777权限 两个证书文件也需要777权限

然后重启nginx即可 重启命令 systemctl restart nginx

至此 搭建完成 访问 https://域名/doc.html 即可

docker镜像会衍生出一个ssh客户端 ip是你的服务器 端口是10800 密码是passwd 请及时更改

```aidl
# For more information on configuration, see:
#   * Official English Documentation: http://nginx.org/en/docs/
#   * Official Russian Documentation: http://nginx.org/ru/docs/

user nginx;
worker_processes auto;
error_log /var/log/nginx/error.log;
pid /run/nginx.pid;

# Load dynamic modules. See /usr/share/doc/nginx/README.dynamic.
include /usr/share/nginx/modules/*.conf;

events {
    worker_connections 1024;
}

http {
    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';

    access_log  /var/log/nginx/access.log  main;

    sendfile            on;
    tcp_nopush          on;
    tcp_nodelay         on;
    keepalive_timeout   65;
    types_hash_max_size 2048;

    include             /etc/nginx/mime.types;
    default_type        application/octet-stream;

    # Load modular configuration files from the /etc/nginx/conf.d directory.
    # See http://nginx.org/en/docs/ngx_core_module.html#include
    # for more information.
    include /etc/nginx/conf.d/*.conf;

	server {
		listen         80;
		server_name    替换域名; # 域名
        client_max_body_size 1024M;
		add_header Strict-Transport-Security max-age=15768000;
		return 301 https://$server_name$request_uri;
    }

	server {
		listen 443 ssl; # 老版本是ssl on;较新的为listen 443 ssl;
		server_name 替换域名; # 域名
		keepalive_timeout 10m;
		client_max_body_size 1024M;
		ssl_certificate      cert/cert.pem; # 申请的证书，把证书和秘钥上传到nginx.conf的同级目录cert的目录下
		ssl_certificate_key  cert/cert.key; # 秘钥
		ssl_session_timeout 24h;
		ssl_ciphers ECDHE-RSA-AES128-GCM-SHA256:ECDHE:ECDH:AES:HIGH:!NULL:!aNULL:!MD5:!ADH:!RC4;
		ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
		ssl_prefer_server_ciphers on;
		location / {
				proxy_pass http://127.0.0.1:10801; # 反向代理到本机的10801端口，10801端口的那个服务器不需要任何关>于https的配置。
				proxy_redirect off;
				proxy_set_header Host $host;
				proxy_set_header X-Real-IP $remote_addr;
				proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
				proxy_set_header X-Forwarded-Proto $scheme;
				proxy_set_header X-Forwarded-Port $server_port;
				add_header Content-Security-Policy upgrade-insecure-requests;
		  }
    }


# Settings for a TLS enabled server.
#
#    server {
#        listen       443 ssl http2 default_server;
#        listen       [::]:443 ssl http2 default_server;
#        server_name  _;
#        root         /usr/share/nginx/html;
#
#        ssl_certificate "/etc/pki/nginx/server.crt";
#        ssl_certificate_key "/etc/pki/nginx/private/server.key";
#        ssl_session_cache shared:SSL:1m;
#        ssl_session_timeout  10m;
#        ssl_ciphers HIGH:!aNULL:!MD5;
#        ssl_prefer_server_ciphers on;
#
#        # Load configuration files for the default server block.
#        include /etc/nginx/default.d/*.conf;
#
#        location / {
#        }
#
#        error_page 404 /404.html;
#        location = /404.html {
#        }
#
#        error_page 500 502 503 504 /50x.html;
#        location = /50x.html {
#        }
#    }

}


```





