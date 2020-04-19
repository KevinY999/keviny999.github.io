---
title: 基于 CentOS 的 ISSO 评论系统后台搭建及修改全程记录
date: 2020-03-06 06:04:10
categories: 博客
tags: [Isso,Nginx,Node,Python3,Supervisor,Hexo]
---

此文用于记录蛋疼的 isso 评论系统配置过程。

<!-- more -->

## 安装环境

1、安装基本环境

```bash
yum -y update && yum install -y wget vim git make automake autoconf gcc gcc-c++ pcre pcre-devel zlib zlib-devel openssl openssl-devel libtool python3 python3-devel
```

2、安装 nginx

```bash
cd ~ \
    && wget http://nginx.org/download/nginx-1.16.1.tar.gz \
    && tar -zxvf nginx-1.16.1.tar.gz \
    && cd nginx-1.16.1 \
    && mkdir -p /usr/local/nginx \
    && ./configure --prefix=/usr/local/nginx --with-http_ssl_module --with-http_stub_status_module --with-http_gzip_static_module --with-pcre \
    && make && make install
```

3、安装 nodejs

```bash
cd ~ \
    && wget https://nodejs.org/dist/v12.16.1/node-v12.16.1-linux-x64.tar.xz \
    && tar xvJf node-v12.16.1-linux-x64.tar.xz \
    && mv node-v12.16.1-linux-x64 /usr/local/node
```

4、清理压缩包

```bash
cd ~ \
    && rm -rf nginx-1.16.1 nginx-1.16.1.tar.gz \
    && rm -rf node-v12.16.1-linux-x64.tar.xz
```

5、设置环境变量

```bash
export NGINX_HOME=/usr/local/nginx
export NODE_HOME=/usr/local/node
export PATH=$NGINX_HOME/sbin:$NODE_HOME/bin:$PATH
```

## 配置并运行 nginx

1、配置 nginx

```bash
vim /usr/local/nginx/conf/nginx.conf
```

**nginx.conf** 文件内容：
```
#user  nobody;
worker_processes  1;

#error_log  logs/error.log;
#error_log  logs/error.log  notice;
#error_log  logs/error.log  info;

pid        logs/nginx.pid;


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

    #gzip  on;

    upstream isso {
        server 127.0.0.1:11006;
    }

    server {
        listen       5000;
        server_name  localhost;

        #charset koi8-r;

        access_log  logs/localhost.access.log;
        error_log   logs/localhost.error.log;

        location /isso {
            proxy_set_header X-Script-Name /isso;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Scheme $scheme;
            proxy_pass http://isso;
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
```

2、运行 nginx

```bash
/usr/local/nginx/sbin/nginx -c /usr/local/nginx/conf/nginx.conf
```

3、重启 nginx

```bash
/usr/local/nginx/sbin/nginx -s reload
```

4、停止 nginx

```bash
/usr/local/nginx/sbin/nginx -s stop
```

## 安装 python 的 virtualenv 模块

```bash
pip3 install virtualenv
```

## 从源码安装 isso

1、将 isso 从 github 克隆到本地

```bash
git clone https://github.com/posativ/isso.git /usr/local/isso
cd /usr/local/isso
```

2、建立 python3 虚拟环境

```bash
virtualenv .
source ./bin/activate
```

3、安装必要 python 第三方库

```bash
pip install flask cffi webencodings six
```

4、安装 isso

```bash
python setup.py develop
```

5、使用 npm 安装 bower

```bash
npm install -g bower
```

6、安装 javascript 模块

```bash
make init
```

7、安装其他几个 js 工具

```bash
npm install -g requirejs uglify-js jade
```

8、修改 isso 源码，以修改即将生成的 **embed.min.js** 文件的内容，实现隐藏 website 输入框、preview 按钮、 edit 按钮以及 preview 界面的目的

```bash
vim isso/js/app/text/postbox.jade
```

为需要修改的部分添加 `style={display: 'none'}` 字符串以隐藏该元素。

**isso/js/app/text/postbox.jade** 文件内容如下：

```
div(class='isso-postbox')
    div(class='form-wrapper')
        div(class='textarea-wrapper')
            div(class='textarea placeholder' contenteditable='true')
                = i18n('postbox-text')
            div(class='preview' style={display: 'none'})
                div(class='isso-comment')
                    div(class='text-wrapper')
                        div(class='text')
        section(class='auth-section')
            p(class='input-wrapper')
                input(type='text' name='author' placeholder=i18n('postbox-author')
                      value=author !== null ? '#{author}' : '')
            p(class='input-wrapper')
                input(type='email' name='email' placeholder=i18n('postbox-email')
                      value=email != null ? '#{email}' : '')
            p(class='input-wrapper' style={display: 'none'})
                input(type='text' name='website' placeholder=i18n('postbox-website')
                      value=website != null ? '#{website}' : '')
            p(class='post-action')
                input(type='submit' value=i18n('postbox-submit'))
            p(class='post-action' style={display: 'none'})
                input(type='button' name='preview'
                      value=i18n('postbox-preview'))
            p(class='post-action' style={display: 'none'})
                input(type='button' name='edit'
                      value=i18n('postbox-edit'))
        section(class='notification-section')
                label
                    input(type='checkbox' name='notification')
                    = i18n('postbox-notification')
```

9、为 isso 添加配置文件 ([文档](https://posativ.org/isso/docs/configuration/server/))

```bash
vim /etc/isso.conf
```

**isso.conf** 文件内容如下：

```
[general]
dbpath = /var/lib/isso/comments.db
host = http://localhost:4000/

[server]
listen = http://localhost:11006/
```

10、创建文件夹

```bash
mkdir -p /var/lib/isso
```

11、生成 js 文件

```bash
make js
```

12、运行 isso

```bash
bin/isso -c /etc/isso.conf
```

## 使用 supervisor 管理 isso

1、使用 pip 工具安装 supervisor

```bash
pip3 install supervisor
```

2、创建文件夹

```bash
mkdir -p /etc/supervisor/conf.d
mkdir -p /var/log/isso
```

3、编写 supervisor 主配置文件

```bash
vim /etc/supervisor/supervisord.conf
```

**supervisord.conf** 文件的内容如下：

```
; Sample supervisor config file.
;
; For more information on the config file, please see:
; http://supervisord.org/configuration.html
;
; Notes:
;  - Shell expansion ("~" or "$HOME") is not supported.  Environment
;    variables can be expanded using this syntax: "%(ENV_HOME)s".
;  - Quotes around values are not supported, except in the case of
;    the environment= options as shown below.
;  - Comments must have a leading space: "a=b ;comment" not "a=b;comment".
;  - Command will be truncated if it looks like a config file comment, e.g.
;    "command=bash -c 'foo ; bar'" will truncate to "command=bash -c 'foo ".
;
; Warning:
;  Paths throughout this example file use /tmp because it is available on most
;  systems.  You will likely need to change these to locations more appropriate
;  for your system.  Some systems periodically delete older files in /tmp.
;  Notably, if the socket file defined in the [unix_http_server] section below
;  is deleted, supervisorctl will be unable to connect to supervisord.

[unix_http_server]
file=/tmp/supervisor.sock   ; the path to the socket file
;chmod=0700                 ; socket file mode (default 0700)
;chown=nobody:nogroup       ; socket file uid:gid owner
;username=user              ; default is no username (open server)
;password=123               ; default is no password (open server)

; Security Warning:
;  The inet HTTP server is not enabled by default.  The inet HTTP server is
;  enabled by uncommenting the [inet_http_server] section below.  The inet
;  HTTP server is intended for use within a trusted environment only.  It
;  should only be bound to localhost or only accessible from within an
;  isolated, trusted network.  The inet HTTP server does not support any
;  form of encryption.  The inet HTTP server does not use authentication
;  by default (see the username= and password= options to add authentication).
;  Never expose the inet HTTP server to the public internet.

;[inet_http_server]         ; inet (TCP) server disabled by default
;port=127.0.0.1:9001        ; ip_address:port specifier, *:port for all iface
;username=user              ; default is no username (open server)
;password=123               ; default is no password (open server)

[supervisord]
logfile=/tmp/supervisord.log ; main log file; default $CWD/supervisord.log
logfile_maxbytes=50MB        ; max main logfile bytes b4 rotation; default 50MB
logfile_backups=10           ; # of main logfile backups; 0 means none, default 10
loglevel=info                ; log level; default info; others: debug,warn,trace
pidfile=/tmp/supervisord.pid ; supervisord pidfile; default supervisord.pid
nodaemon=false               ; start in foreground if true; default false
minfds=1024                  ; min. avail startup file descriptors; default 1024
minprocs=200                 ; min. avail process descriptors;default 200
;umask=022                   ; process file creation umask; default 022
;user=supervisord            ; setuid to this UNIX account at startup; recommended if root
;identifier=supervisor       ; supervisord identifier, default is 'supervisor'
;directory=/tmp              ; default is not to cd during start
;nocleanup=true              ; don't clean up tempfiles at start; default false
;childlogdir=/tmp            ; 'AUTO' child log dir, default $TEMP
;environment=KEY="value"     ; key value pairs to add to environment
;strip_ansi=false            ; strip ansi escape codes in logs; def. false

; The rpcinterface:supervisor section must remain in the config file for
; RPC (supervisorctl/web interface) to work.  Additional interfaces may be
; added by defining them in separate [rpcinterface:x] sections.

[rpcinterface:supervisor]
supervisor.rpcinterface_factory = supervisor.rpcinterface:make_main_rpcinterface

; The supervisorctl section configures how supervisorctl will connect to
; supervisord.  configure it match the settings in either the unix_http_server
; or inet_http_server section.

[supervisorctl]
serverurl=unix:///tmp/supervisor.sock ; use a unix:// URL  for a unix socket
;serverurl=http://127.0.0.1:9001 ; use an http:// url to specify an inet socket
;username=chris              ; should be same as in [*_http_server] if set
;password=123                ; should be same as in [*_http_server] if set
;prompt=mysupervisor         ; cmd line prompt (default "supervisor")
;history_file=~/.sc_history  ; use readline history if available

; The sample program section below shows all possible program subsection values.
; Create one or more 'real' program: sections to be able to control them under
; supervisor.

;[program:theprogramname]
;command=/bin/cat              ; the program (relative uses PATH, can take args)
;process_name=%(program_name)s ; process_name expr (default %(program_name)s)
;numprocs=1                    ; number of processes copies to start (def 1)
;directory=/tmp                ; directory to cwd to before exec (def no cwd)
;umask=022                     ; umask for process (default None)
;priority=999                  ; the relative start priority (default 999)
;autostart=true                ; start at supervisord start (default: true)
;startsecs=1                   ; # of secs prog must stay up to be running (def. 1)
;startretries=3                ; max # of serial start failures when starting (default 3)
;autorestart=unexpected        ; when to restart if exited after running (def: unexpected)
;exitcodes=0                   ; 'expected' exit codes used with autorestart (default 0)
;stopsignal=QUIT               ; signal used to kill process (default TERM)
;stopwaitsecs=10               ; max num secs to wait b4 SIGKILL (default 10)
;stopasgroup=false             ; send stop signal to the UNIX process group (default false)
;killasgroup=false             ; SIGKILL the UNIX process group (def false)
;user=chrism                   ; setuid to this UNIX account to run the program
;redirect_stderr=true          ; redirect proc stderr to stdout (default false)
;stdout_logfile=/a/path        ; stdout log path, NONE for none; default AUTO
;stdout_logfile_maxbytes=1MB   ; max # logfile bytes b4 rotation (default 50MB)
;stdout_logfile_backups=10     ; # of stdout logfile backups (0 means none, default 10)
;stdout_capture_maxbytes=1MB   ; number of bytes in 'capturemode' (default 0)
;stdout_events_enabled=false   ; emit events on stdout writes (default false)
;stdout_syslog=false           ; send stdout to syslog with process name (default false)
;stderr_logfile=/a/path        ; stderr log path, NONE for none; default AUTO
;stderr_logfile_maxbytes=1MB   ; max # logfile bytes b4 rotation (default 50MB)
;stderr_logfile_backups=10     ; # of stderr logfile backups (0 means none, default 10)
;stderr_capture_maxbytes=1MB   ; number of bytes in 'capturemode' (default 0)
;stderr_events_enabled=false   ; emit events on stderr writes (default false)
;stderr_syslog=false           ; send stderr to syslog with process name (default false)
;environment=A="1",B="2"       ; process environment additions (def no adds)
;serverurl=AUTO                ; override serverurl computation (childutils)

; The sample eventlistener section below shows all possible eventlistener
; subsection values.  Create one or more 'real' eventlistener: sections to be
; able to handle event notifications sent by supervisord.

;[eventlistener:theeventlistenername]
;command=/bin/eventlistener    ; the program (relative uses PATH, can take args)
;process_name=%(program_name)s ; process_name expr (default %(program_name)s)
;numprocs=1                    ; number of processes copies to start (def 1)
;events=EVENT                  ; event notif. types to subscribe to (req'd)
;buffer_size=10                ; event buffer queue size (default 10)
;directory=/tmp                ; directory to cwd to before exec (def no cwd)
;umask=022                     ; umask for process (default None)
;priority=-1                   ; the relative start priority (default -1)
;autostart=true                ; start at supervisord start (default: true)
;startsecs=1                   ; # of secs prog must stay up to be running (def. 1)
;startretries=3                ; max # of serial start failures when starting (default 3)
;autorestart=unexpected        ; autorestart if exited after running (def: unexpected)
;exitcodes=0                   ; 'expected' exit codes used with autorestart (default 0)
;stopsignal=QUIT               ; signal used to kill process (default TERM)
;stopwaitsecs=10               ; max num secs to wait b4 SIGKILL (default 10)
;stopasgroup=false             ; send stop signal to the UNIX process group (default false)
;killasgroup=false             ; SIGKILL the UNIX process group (def false)
;user=chrism                   ; setuid to this UNIX account to run the program
;redirect_stderr=false         ; redirect_stderr=true is not allowed for eventlisteners
;stdout_logfile=/a/path        ; stdout log path, NONE for none; default AUTO
;stdout_logfile_maxbytes=1MB   ; max # logfile bytes b4 rotation (default 50MB)
;stdout_logfile_backups=10     ; # of stdout logfile backups (0 means none, default 10)
;stdout_events_enabled=false   ; emit events on stdout writes (default false)
;stdout_syslog=false           ; send stdout to syslog with process name (default false)
;stderr_logfile=/a/path        ; stderr log path, NONE for none; default AUTO
;stderr_logfile_maxbytes=1MB   ; max # logfile bytes b4 rotation (default 50MB)
;stderr_logfile_backups=10     ; # of stderr logfile backups (0 means none, default 10)
;stderr_events_enabled=false   ; emit events on stderr writes (default false)
;stderr_syslog=false           ; send stderr to syslog with process name (default false)
;environment=A="1",B="2"       ; process environment additions
;serverurl=AUTO                ; override serverurl computation (childutils)

; The sample group section below shows all possible group values.  Create one
; or more 'real' group: sections to create "heterogeneous" process groups.

;[group:thegroupname]
;programs=progname1,progname2  ; each refers to 'x' in [program:x] definitions
;priority=999                  ; the relative start priority (default 999)

; The [include] section can just contain the "files" setting.  This
; setting can list multiple files (separated by whitespace or
; newlines).  It can also contain wildcards.  The filenames are
; interpreted as relative to this file.  Included files *cannot*
; include files themselves.

[include]
files = /etc/supervisor/conf.d/*.conf
```

4、编辑 supervisor 对于 isso 的专属配置文件

```bash
vim /etc/supervisor/conf.d/isso.conf
```

**isso.conf** 文件的内容如下：

```
[program:isso]
command=/usr/local/isso/bin/isso -c /etc/isso.conf run
directory=/var/lib/isso
user=root
autorestart=true
stdout_logfile=/var/log/isso/access.log
stderr_logfile=/var/log/isso/error.log
loglevel=info
environment=LANG="en_US.UTF-8"
```

5、启动 supervisord 服务

```bash
supervisord -c /etc/supervisor/supervisord.conf
```

6、管理 supervisor 中的 isso

**启动**: `supervisorctl start isso`

**重启**: `supervisorctl restart isso`

**停止**: `supervisorctl stop isso`

## 访问 isso 服务

```bash
curl http://localhost:5000/isso
```

## 修改 hexo 下 icarus 主题

1、增加 isso.css 样式表文件

```bash
vim themes/icarus/source/css/isso.css
```

**isso.css** 文件的内容如下：

```
#isso-thread h4 {
    margin: 1em 0;
    font-weight: bold;
    font-size: 1.5em;
}
.isso-postbox > .form-wrapper > .textarea-wrapper > .textarea {
    width: 100%;
    height: auto;
    min-height: 8em;
    overflow-y: scroll;
    resize: none;
    outline: none;
    -webkit-user-modify: read-write-plaintext-only;
}
.isso-postbox > .form-wrapper > .auth-section {
    width: 100%;
    height: 3rem;
    display: -webkit-flex;
    display: flex;
    justify-content: space-between;
    margin: .6rem 0;
}
.isso-postbox > .form-wrapper > .auth-section p {
    flex: 1;
}
.isso-postbox > .form-wrapper > .auth-section > .input-wrapper {
    margin-right: .6rem;
}
.isso-postbox > .form-wrapper > .auth-section > p > input {
    width: 100%;
    height: 100%;
    display: block;
    padding: .6rem .8rem;
    font-size: 14px;
    border-radius: .2rem;
    outline: none;
}
.isso-postbox > .form-wrapper > .auth-section > .input-wrapper  input {
    background: #fafafa;
    border: .1rem solid #eee;
    cursor: text;
}
.isso-postbox > .form-wrapper > .auth-section > .post-action  input {
    background: #2687fb;
    border: .1rem solid #177ffb;
    color: #fff;
    cursor: pointer;
}
.isso-postbox > .form-wrapper > .notification-section input {
    margin-right: .3rem;
}
#isso-root .isso-comment {
    max-width: 68em;
    padding-top: 0.95em;
    margin: 0.95em auto;
}
.isso-comment > div.avatar {
    display: block;
    float: left;
    width: 7%;
    margin: 3px 15px 0 0;
}
.isso-comment > div.avatar > svg {
    max-width: 48px;
    max-height: 48px;
    border: 1px solid rgba(0, 0, 0, 0.2);
    border-radius: 3px;
    box-shadow: 0 1px 2px rgba(0, 0, 0, 0.1);
}

.isso-comment > div.text-wrapper {
    display: block;
}
.isso-comment .isso-follow-up {
    padding-left: calc(7% + 20px);
}
.isso-comment > div.text-wrapper > .isso-comment-header, .isso-comment > div.text-wrapper > .isso-comment-footer {
    font-size: 0.95em;
}
.isso-comment > div.text-wrapper > .isso-comment-header {
    font-size: 0.85em;
}
.isso-comment > div.text-wrapper > .isso-comment-header .spacer {
    padding: 0 6px;
}
.isso-comment > div.text-wrapper > .isso-comment-header .spacer,
.isso-comment > div.text-wrapper > .isso-comment-header a.permalink,
.isso-comment > div.text-wrapper > .isso-comment-header .note,
.isso-comment > div.text-wrapper > .isso-comment-header a.parent {
    color: gray !important;
    font-weight: normal;
    text-shadow: none !important;
}
.isso-comment > div.text-wrapper > .isso-comment-header .spacer:hover,
.isso-comment > div.text-wrapper > .isso-comment-header a.permalink:hover,
.isso-comment > div.text-wrapper > .isso-comment-header .note:hover,
.isso-comment > div.text-wrapper > .isso-comment-header a.parent:hover {
    color: #606060 !important;
}
.isso-comment > div.text-wrapper > .isso-comment-header .note {
    float: right;
}
.isso-comment > div.text-wrapper > .isso-comment-header .author {
    font-weight: bold;
    color: #555;
}
.isso-comment > div.text-wrapper > .textarea-wrapper .textarea,
.isso-comment > div.text-wrapper > .textarea-wrapper .preview {
    margin-top: 0.2em;
}
.isso-comment > div.text-wrapper > div.text p {
    margin-top: 0.2em;
}
.isso-comment > div.text-wrapper > div.text p:last-child {
    margin-bottom: 0.2em;
}
.isso-comment > div.text-wrapper > div.text h1,
.isso-comment > div.text-wrapper > div.text h2,
.isso-comment > div.text-wrapper > div.text h3,
.isso-comment > div.text-wrapper > div.text h4,
.isso-comment > div.text-wrapper > div.text h5,
.isso-comment > div.text-wrapper > div.text h6 {
    font-size: 130%;
    font-weight: bold;
}
.isso-comment > div.text-wrapper > div.textarea-wrapper .textarea,
.isso-comment > div.text-wrapper > div.textarea-wrapper .preview {
    width: 100%;
    border: 1px solid #f0f0f0;
    border-radius: 2px;
    box-shadow: 0 0 2px #888;
}
.isso-comment > div.text-wrapper > .isso-comment-footer {
    font-size: 0.80em;
    color: gray !important;
    clear: left;
}
.isso-feedlink,
.isso-comment > div.text-wrapper > .isso-comment-footer a {
    font-weight: bold;
    text-decoration: none;
}
.isso-feedlink:hover,
.isso-comment > div.text-wrapper > .isso-comment-footer a:hover {
    color: #111111 !important;
    text-shadow: #aaaaaa 0 0 1px !important;
}
.isso-comment > div.text-wrapper > .isso-comment-footer > a {
    position: relative;
    top: .2em;
}
.isso-comment > div.text-wrapper > .isso-comment-footer > a + a {
    padding-left: 1em;
}
.isso-comment > div.text-wrapper > .isso-comment-footer .votes {
    color: gray;
}
.isso-comment > div.text-wrapper > .isso-comment-footer .upvote svg,
.isso-comment > div.text-wrapper > .isso-comment-footer .downvote svg {
    position: relative;
    top: .2em;
}
.isso-comment .isso-postbox {
    margin-top: 0.8em;
}
.isso-comment.isso-no-votes span.votes {
    display: none;
}
```

2、修改 isso.ejs 这一 isso 布局文件

```bash
vim themes/icarus/layout/comment/isso.ejs
```

**isso.ejs** 文件的内容如下：

```
<% if (!has_config('comment.url')) { %>
<div class="notification is-danger">
    You forgot to set the <code>url</code> for Isso. Please set it in <code>_config.yml</code>.
</div>
<% } else { %>
<%- _css('css/isso') %>
<script data-isso="//<%= get_config('comment.url') %>"
    data-isso-css="false"
    data-isso-lang="zh"
    data-isso-reply-to-self="true"
    data-isso-require-author="true"
    data-isso-reply-notifications="true"
    src="//<%= get_config('comment.url') %>/js/embed.min.js">
</script>
<section id="isso-thread" data-title="<%= post.title %>"></section>
<% } %>
```

3、修改 icarus 主题的 _config.yml 文件，添加 isso 的相关配置 ([文档](https://blog.zhangruipeng.me/hexo-theme-icarus/categories/Plugins/Comment/))
