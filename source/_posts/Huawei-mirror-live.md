---
title: 基于Airserver的Huawei手机简易投屏直播服务搭建过程全记录
date: 2019-10-22 13:40:16
categories: 多媒体
tags: [Huawei,Nginx,Rtmp,Airserver]
---

2019 年 7 月 2 日，是一个令无数跑跑卡丁车端游老玩家为之兴奋的日子，腾讯游戏联合世纪天成推出了跑跑的手游并正式开服，开服前的预约人数就已经突破了 2000 万。作为一个当年端游的死忠粉，当然也要赶上这波热潮。到今天为止，跑跑手游已经运营了快四个月了，这期间推出了无数的人物和赛车，上新的速度远超氪金的速度括弧笑。喜欢玩游戏的朋友应该大多有为了体验新人物、新道具或者帮人上分而使用好友账号登陆的经历，这不，贫穷的我想用一个土豪朋友的号体验一下新出的光明骑士。

腾讯为游戏划分了微信、QQ、苹果、安卓四种区服，我的土豪朋友是在安卓微信区，那么我需要使用微信登陆上他的游戏账号。说到微信登陆，真的是既方便又不方便，方便的是它提供了扫码登录的功能，朋友不需要告知我他的用户名和密码便可以安全的让我登录上他的账号。不方便的地方是，游戏必须在没有检测到设备已安装微信 app 的情况下才会提供扫码登录的功能，而且据说从微信 7.0 版本开始，截图用于登录的二维码再从相册导入截图的扫码方法已经失效，只能当面扫一扫。然鹅我和我的土豪朋友相隔一千多公里，为了解决这一问题，有了下面的一大波踩坑和填坑的辛酸泪。

<!-- more -->

**我的设备：1 台 Huawei Mate10 Pro手机，1 台 Mac 电脑，1 台闲置的服务器。**

为了实现远距离的扫码登录，参考各大直播平台主播代练的方法，我决定也使用直播这种方式来完成这波操作。但问题是**贫穷**限制了我只有一台手机，并且已经安装了微信 app ，如何在不删除现有微信的情况下建立一个没有微信 app 的环境成为一个问题；并且我也不想为此就去某个平台注册一个主播号，因为太麻烦，那只能自己手动搭建一个直播环境了。**大致的思路就是：利用一个独立的无微信的空间，在其中安装跑跑手游 app 并打开二维码登录界面，将手机投屏到我的 Mac 电脑端，通过 OBS 工具推流到服务器，我的好友访问对应的直播地址进行观看并扫码。**思路大概有了，那么就去实践一下吧。

![](https://cdn.jsdelivr.net/gh/KevinY999-pic/Kev1nBlog/vaylf.png)

## 填坑一：在一台手机上建立有微信 app 和没有微信 app 的两种环境

+ **方法 1 **：不知什么时候开始，时常能在小红书、抖音、微博上看到大吹华为手机自带的**隐私空间**功能的短视频，官方介绍为：隐私空间是一个可存储私人数据、**独立**于主空间的私密空间。面对第一个问题，我第一反应便是这个。具体的开启路径为 [设置] -> [安全和隐私] -> [隐私空间] -> [开启] 。

![华为隐私空间](https://cdn.jsdelivr.net/gh/KevinY999-pic/Kev1nBlog/kc7j7.jpg)

+ **方法 2 **：当然，创建另一个独立空间的方法肯定不止隐私空间这一种，另一个可行的方法是，利用华为的**多账户功能**。具体设置路径为 [设置] -> [用户和账户] -> [多用户] -> [添加用户] 。

![华为多账户](https://cdn.jsdelivr.net/gh/KevinY999-pic/Kev1nBlog/o8iif.png)

新建账户之后需要进行设置，跳过网络连接的步骤便无需登录华为账户，设置完成后，可以通过点击通知栏上方的头像切换回主账户。

![账户切换1](https://cdn.jsdelivr.net/gh/KevinY999-pic/Kev1nBlog/xxh0u.png)

![账户切换2](https://cdn.jsdelivr.net/gh/KevinY999-pic/Kev1nBlog/4naij.png)

## 填坑二：将华为手机投屏到 Mac 端

这里采用一个牛逼的工具 —— [**AirServer for MacOS**](https://www.airserver.com/Mac) 。

![AirServer-logo](https://cdn.jsdelivr.net/gh/KevinY999-pic/Kev1nBlog/6x4ub.png)

引用官方的介绍：AirServer is the most advanced screen mirroring receiver for Mac and PC. It allows you to receive AirPlay and Google Cast streams, similar to an Apple TV or a Chromecast device.

AirServer 是一款收费软件，Mac 版有三个价格，当然也可以免费试用 14 天：

![AirServer for MacOS 价格](https://cdn.jsdelivr.net/gh/KevinY999-pic/Kev1nBlog/b8457.png)

网上有很多的破解版，不过还是建议大家购买正版，支持一下这款非常优秀的软件，我先买为敬。(官方支持信用卡及 PayPal 付款)

AirServer for MacOS 并没有可视化的主界面，只有设置界面：

![AirServer 设置界面截图](https://cdn.jsdelivr.net/gh/KevinY999-pic/Kev1nBlog/6mfnj.png)

这里几乎无需设置，只需要设置一下计算机名称便可直接使用。

电脑端配置完成，还需要手机的连接才行。苹果手机用自带的 AirPlay 便可以直接投屏到 Mac 端，而安卓手机需要额外使用的应用。这里使用官方的 [AirServer Connect](https://play.google.com/store/apps/details?id=com.appdynamic.airserverconnect&hl=en) 或者 Google 推出的 [Google Home](https://play.google.com/store/apps/details?id=com.google.android.apps.chromecast.app&hl=en) 两个应用程序进行连接，它们都需要从 Google Play 商店进行下载。

> **基本连接环境：手机与电脑需要处于同一个 Wi-Fi 环境下。**

### AirServer Connect 连接方法

1、点击 Mac 端最上方菜单栏中 AirServer 的图标，选择  `Show QR Code` 选项，如图：

![打开 AirServer 连接二维码](https://cdn.jsdelivr.net/gh/KevinY999-pic/Kev1nBlog/yu73v.png)

2、打开手机端的 AirServer Connect 应用程序，点击上方的二维码图标，扫码 Mac 端二维码进行连接，如图：

![AirServer Connect 扫码连接](https://cdn.jsdelivr.net/gh/KevinY999-pic/Kev1nBlog/8tv19.png)

当然，如果知道 Mac 端的局域网地址，也可以在 AirServer Connect 应用程序右上角的菜单栏中点击`连接到...` ，输入 Mac 端的局域网地址进行快速连接，如图：

![选择"连接到..."选项](https://cdn.jsdelivr.net/gh/KevinY999-pic/Kev1nBlog/2b3v0.png)

![输入 Mac 端地址](https://cdn.jsdelivr.net/gh/KevinY999-pic/Kev1nBlog/j0whs.png)

连接完成后，Mac 端会弹出手机投屏的对话框，如图：

![AirServer Connect 连接成功](https://cdn.jsdelivr.net/gh/KevinY999-pic/Kev1nBlog/utal0.png)

### Google Home 连接方法

1、在 Mac 端的 AirServer 设置中勾选 `Enable Google Cast` ，如图：

![](https://cdn.jsdelivr.net/gh/KevinY999-pic/Kev1nBlog/5ubdf.png)

2、打开 Google Home 应用程序，切换到最后的![](https://cdn.jsdelivr.net/gh/KevinY999-pic/Kev1nBlog/y7s6z.png)`账户信息`标签页；

3、向下滑动选择`镜像设备内容`选项，如图：

![](https://cdn.jsdelivr.net/gh/KevinY999-pic/Kev1nBlog/3lzvi.png)

4、进入镜像设备内容后，点击`投射屏幕/音频`按钮，程序会自动搜索同一局域网络下支持投射的设备，如图：

![](https://cdn.jsdelivr.net/gh/KevinY999-pic/Kev1nBlog/luuvu.png)

5、选择要投射到的设备，如图：

![](https://cdn.jsdelivr.net/gh/KevinY999-pic/Kev1nBlog/rq5mj.png)

### 两种投屏方式比较

1、投屏功能：二者都能够将手机画面投射到 Mac 端，但出于谷歌某些安全机制原因，AirServer Connect 无法将手机设备的声音也投射过去，而 Google Home 可以，具体原因如图：

![AirServer Connect 无法投射声音原因](https://cdn.jsdelivr.net/gh/KevinY999-pic/Kev1nBlog/3vzek.png)

2、投屏效率：作为 AirServer 官方出品的手机客户端，AirServer Connect 当然能够更高效地将画面投射到客户端，而 AirServer 同样兼容的 Google Cast 功能则效率相对会低一点，在打开 Goolge Home 的投屏功能时，该应用程序也会弹出一个提示框，如图：

![Google Cast设备未优化警告](https://cdn.jsdelivr.net/gh/KevinY999-pic/Kev1nBlog/jtck5.png)

这种情况下，经过实测，画面可能会有少许掉帧，没有 AirServer Connect 那么流畅，但是能将声音投射给 Mac 端，还是能接受的。

## 踩坑一：AirServer Connect 连接 Mac 端后一分钟内连接会自动断开

这是由于华为手机自带的手机管家默认对应用启动权限进行了自动托管，只需要对 AirServer Connect 这一应用关闭自动托管功能就好，具体操作方法为：[手机管家] -> [应用启动管理] -> [将 AirServer Connect 后的开关拨至手动管理状态] -> [在弹出的窗口中将三个开关都拨至打开状态]。如图：

![](https://cdn.jsdelivr.net/gh/KevinY999-pic/Kev1nBlog/y82pj.png)

![](https://cdn.jsdelivr.net/gh/KevinY999-pic/Kev1nBlog/6wbgx.png)

这样，AirServer Connect 应用程序就能保持在后台运行不会被安卓系统自动回收了。

## 踩坑二：华为手机子账户或隐私空间中不可管理应用启动权限

在华为手机的子账户或隐私空间中，手机管家这一应用程序中，只有流量管理和电池管理两个选项，其余部分都无法进入，如图：

![](https://cdn.jsdelivr.net/gh/KevinY999-pic/Kev1nBlog/ozaap.png)

解决方法：进入主账户的手机管家中进行设置，**子账户中的权限将与主账户中设置的同步**。

> 注意：**请使用华为子账户而非隐私空间**，经测试，隐私空间中应用的权限极低，并且不与主账户中设置的权限同步。

---

华为手机投屏的坑都踩的差不多了，下面该介绍一下简单的直播服务的搭建过程了。

## NGINX 服务器程序的编译与安装

> 官方介绍：nginx [engine x] is an HTTP and reverse proxy server, a mail proxy server, and a generic TCP/UDP proxy server, originally written by [Igor Sysoev](http://sysoev.ru/en/).

为了实现高质量低延迟的实时在线直播功能，决定采用 rtmp 协议，并为 nginx 添加 rtmp 插件，这里采用开源的 nginx-rtmp-module 插件，GitHub 地址：https://github.com/arut/nginx-rtmp-module。

为了给 nginx 安装上 rtmp 插件，需要将 nginx 从源码编译，并在参数中添加上 rtmp 模块。以 CentOS 7 系统为例：

+ 更新 yum 仓库并安装必要工具：

  ```bash
  yum -y update && yum install -y wget git make automake autoconf gcc gcc-c++ pcre pcre-devel zlib zlib-devel openssl openssl-devel libtool
  ```

+ 下载 nginx 源码并克隆 nginx-rtmp-module 仓库，编译安装：

  ```bash
  cd /tmp && git clone https://github.com/arut/nginx-rtmp-module.git \
  		&& wget http://nginx.org/download/nginx-1.16.1.tar.gz \
      && tar -zxvf nginx-1.16.1.tar.gz \
      && cd nginx-1.16.1 \
      && mkdir -p /usr/local/nginx \
      && ./configure --prefix=/usr/local/nginx --with-http_ssl_module --with-http_stub_status_module --with-http_gzip_static_module --with-pcre --add-module=/nginx-rtmp-module \
      && make && make install \
      && cd /tmp && rm -rf nginx-rtmp-module nginx-1.16.1 nginx-1.16.1.tar.gz
  ```

这样就成功安装了支持 rtmp 的 nginx 服务器程序。

## NGINX 的配置

配置文件路径为 nginx 根目录下 conf/nginx.conf 。

在该文件中添加 rtmp 部分的配置，最简单的配置内容如下：

```
rtmp {
    server {
        listen 1935;  # 默认监听端口
        chunk_size 4000;  # 默认数据传输块大小
        application live {  # application后的名称为rtmp推流的请求路径
            live on;  # 开启直播
        }
    }
}
```

有了以上配置，在启动 nginx 服务器后就可以通过支持 rtmp 流的播放器查看直播内容了，或者使用 ffplay 直接播放，地址为：`rtmp://<IP>:1935/live` ，播放命令为：`ffplay -i "rtmp://<IP>:1935/live"` 。

对于一些移动端的朋友，可能这样收看直播并不方便，更为直观和简便的方法是在网页中嵌入一个播放器，在此播放器中播放直播的媒体流。最简单的方案是开启 hls 支持，那么我们就为 nginx 添加相关配置。

```
rtmp {
    server {
        listen 1935;
        chunk_size 4000;
        application live {
            live on;
            hls on;  # 开启hls
            hls_path html/hls;  # 切片文件存放路径
            hls_fragment 5s;  # 每个切片文件包含的视频时长为5秒
            hls_playlist_length 15s;  # hls播放列表的长度
            hls_continuous on;  # 连续模式
            hls_cleanup on;  # 对多余的切片进行删除
            hls_nested on;  # 嵌套模式
        }
    }
}
```

为了能够以 http 协议获取 hls ，还需要在配置文件的 http 模块下添加如下的配置：

```
server {
    listen 8888;
    server_name localhost;

    location /player {
        alias /usr/local/nginx/html/;  # 嵌入播放器的html文件路径
        index player.html;
    }

    location /hls-live {
        alias /usr/local/nginx/html/hls/;  # 切片文件路径
        types {
            application/vnd.apple.mpegurl m3u8;
            video/mp2t ts;
        }
        expires -1;
        add_header Cache-Control no-cache;
        add_header 'Access-Control-Allow-Origin' '*';
        add_header 'Access-Control-Allow-Credentials' 'true';
        add_header 'Access-Control-Allow-Methods' 'GET, POST, OPTIONS';
        add_header 'Access-Control-Allow-Headers' 'DNT,X-CustomHeader,Keep-Alive,User-Agent,X-Requested-With,If-Modified-Since,Cache-Control,Content-Type';
    }
}
```

完整的 `nginx.conf` 文件内容：

```

#user  nobody;
worker_processes  1;

#error_log  logs/error.log;
#error_log  logs/error.log  notice;
#error_log  logs/error.log  info;

#pid        logs/nginx.pid;


events {
    worker_connections  1024;
}


http {
    include       mime.types;
    default_type  application/octet-stream;

    #log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
    #                  '$status $body_bytes_sent "$http_referer" '
    #                  '"$http_user_agent" "$http_x_forwarded_for"';

    #access_log  logs/access.log  main;

    sendfile        on;
    #tcp_nopush     on;

    #keepalive_timeout  0;
    keepalive_timeout  65;

    gzip  on;

    server {
        listen       80;
        server_name  localhost;

        #charset koi8-r;

        #access_log  logs/host.access.log  main;

        location / {
            root   html;
            index  index.html index.htm;
        }

        #error_page  404              /404.html;

        # redirect server error pages to the static page /50x.html
        #
        error_page   500 502 503 504  /50x.html;
        location = /50x.html {
            root   html;
        }

        # proxy the PHP scripts to Apache listening on 127.0.0.1:80
        #
        #location ~ \.php$ {
        #    proxy_pass   http://127.0.0.1;
        #}

        # pass the PHP scripts to FastCGI server listening on 127.0.0.1:9000
        #
        #location ~ \.php$ {
        #    root           html;
        #    fastcgi_pass   127.0.0.1:9000;
        #    fastcgi_index  index.php;
        #    fastcgi_param  SCRIPT_FILENAME  /scripts$fastcgi_script_name;
        #    include        fastcgi_params;
        #}

        # deny access to .htaccess files, if Apache's document root
        # concurs with nginx's one
        #
        #location ~ /\.ht {
        #    deny  all;
        #}
    }

    server {
        listen 8888;
        server_name localhost;

        location /player {
            alias /usr/local/nginx/html/;
            index player.html;
        }

        location /hls-live {
            alias /usr/local/nginx/html/hls/;
            types {
                application/vnd.apple.mpegurl m3u8;
                video/mp2t ts;
            }
            expires -1;
            add_header Cache-Control no-cache;
            add_header 'Access-Control-Allow-Origin' '*';
            add_header 'Access-Control-Allow-Credentials' 'true';
            add_header 'Access-Control-Allow-Methods' 'GET, POST, OPTIONS';
            add_header 'Access-Control-Allow-Headers' 'DNT,X-CustomHeader,Keep-Alive,User-Agent,X-Requested-With,If-Modified-Since,Cache-Control,Content-Type';
        }
    }


    # another virtual host using mix of IP-, name-, and port-based configuration
    #
    #server {
    #    listen       8000;
    #    listen       somename:8080;
    #    server_name  somename  alias  another.alias;

    #    location / {
    #        root   html;
    #        index  index.html index.htm;
    #    }
    #}


    # HTTPS server
    #
    #server {
    #    listen       443 ssl;
    #    server_name  localhost;

    #    ssl_certificate      cert.pem;
    #    ssl_certificate_key  cert.key;

    #    ssl_session_cache    shared:SSL:1m;
    #    ssl_session_timeout  5m;

    #    ssl_ciphers  HIGH:!aNULL:!MD5;
    #    ssl_prefer_server_ciphers  on;

    #    location / {
    #        root   html;
    #        index  index.html index.htm;
    #    }
    #}

}

rtmp {
    server {
        listen 1935;
        chunk_size 4000;
        application live {
            live on;
            hls on;
            hls_path html/hls;
            hls_fragment 5s;
            hls_playlist_length 15s;
            hls_continuous on;
            hls_cleanup on;
            hls_nested on;
        }
    }
}

```

## 编写支持 hls 流媒体播放的静态 html 页面

如上面 nginx.conf 中的配置，在 `/usr/local/nginx/html/` 路径下创建 `player.html` 文件。这里采用 **`video.js`** 以播放 hls 流，文件内容如下：

```html
<!DOCTYPE html>
<html>
    <head>
        <meta charset="UTF-8">
        <meta name="viewport" content="width=device-width, user-scalable=no, initial-scale=1.0, maximum-scale=1.0, minimum-scale=1.0">
        <meta http-equiv="X-UA-Compatible" content="ie=edge">
        <title>Live Stream</title>
        <link href="https://cdn.bootcss.com/video.js/7.6.5/video-js.css" rel="stylesheet">
        <script src="https://cdn.bootcss.com/video.js/7.6.5/video.min.js"></script>
        <script src="https://cdn.bootcss.com/videojs-contrib-hls/5.15.0/videojs-contrib-hls.js"></script>
        <style type="text/css">
            html, body {
                width: 100%;
                height: 100%;
                margin: 0 auto;
            }
        </style>
    </head>
    <body>
        <video id="live-video" style="width: 100%; height: 100%;" class="video-js vjs-default-skin vjs-big-play-centered" controls poster="">
            <source src="http://<IP>:8888/hls-live/index.m3u8" type="application/x-mpegURL">
        </video>
        <script type="text/javascript">
            var player = videojs('live-video');
            player.play();
        </script>
    </body>
</html>

```

## NGINX 的启动、停止、重新加载

**启动**：`/usr/local/nginx/sbin/nginx -c /usr/local/nginx/conf/nginx.conf`

**停止**：`/usr/local/nginx/sbin/nginx -s stop`

**重新加载**：`/usr/local/nginx/sbin/nginx -s reload`

## CentOS 7 防火墙设置

这里使用 `firewall-cmd` 命令进行管理。

### firewalld 服务基本命令

**启动**：`systemctl start firewalld`

**停止**：`systemctl stop firewalld`

**查看状态**：`systemctl status firewalld`

### firewall-cmd 命令基本使用

**查看防火墙状态**：`firewall-cmd --state`

**查看防火墙所有端口情况**：`firewall-cmd --list-all`

**开启某个端口**：`firewall-cmd --permanent --add-port=80/tcp`

**关闭某个端口**：`firewall-cmd --permanent --remove-port=80/tcp`

**重载防火墙配置**：`firewall-cmd --reload`

> 注意：在开启或关闭某个/些端口后需要重载防火墙配置才能生效。

## OBS 推流设置

打开 OBS 工具的设置界面，选择左侧的 `推流` 标签页，如图：

![进行推流设置](https://cdn.jsdelivr.net/gh/KevinY999-pic/Kev1nBlog/58a0a.png)

在右侧 `服务器` 部分填入上面 nginx.conf 中设置的 rtmp 推流地址，如图：

![填入推流地址](https://cdn.jsdelivr.net/gh/KevinY999-pic/Kev1nBlog/4wzri.png)

设置完成后，回到 OBS 主界面，添加`Syphon客户端`，选择 AirServer 的投屏窗口，然后在右侧点击 `开始推流` ，稍等片刻，刷新我们的播放器界面，便能收看到直播的内容了。

## 关于 MAC 内录声音的问题

苹果出于防盗版的考虑，并不支持 mac 的视频或者声音内录，这就需要用到一个开源神器，[**SoundFlower**](https://github.com/mattingalls/Soundflower) 。

在安装完成该插件后，进入到 `应用程序` ，打开 `音频 MIDI 设置` ，如图：

![音频MIDI设置](https://cdn.jsdelivr.net/gh/KevinY999-pic/Kev1nBlog/htkob.png)

进入设置后，点击左下方的 `+` 号按钮，选择第二项 `多输出设备` ，在右侧勾选 `内建输出` 和 `Soundflower (2ch)` 选项，如图：

![新建多输出设备](https://cdn.jsdelivr.net/gh/KevinY999-pic/Kev1nBlog/fm1xt.png)

这样，就可以在保持监听的同时，将 mac 内播放的声音录入，可以在使用 OBS 工具直播的同时播放音乐，增加直播的乐趣。

