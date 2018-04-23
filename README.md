nginx+lua构建简单waf网页防火墙
需求背景

类似于论坛型的网站经常会被黑掉，除了增加硬件防护感觉效果还是不太好，还会偶尔被黑，waf的功能正好实现了这个需求。

waf的作用：

防止sql注入，本地包含，部分溢出，fuzzing测试，xss,SSRF等web攻击
防止svn/备份之类文件泄漏
防止ApacheBench之类压力测试工具的攻击
屏蔽常见的扫描黑客工具，扫描器
屏蔽异常的网络请求
屏蔽图片附件类目录php执行权限
防止webshell上传
nginx 的话我选择春哥开源的：OpenResty一个伟大的项目。

OpenResty 介绍

OpenResty(又称：ngx_openresty) 是一个基于 NGINX 的可伸缩的 Web 平台，由中国人章亦春发起，提供了很多高质量的第三方模块。

OpenResty 是一个强大的 Web 应用服务器，Web 开发人员可以使用 Lua 脚本语言调动 Nginx 支持的各种 C 以及 Lua 模块,更主要的是在性能方面，OpenResty可以 快速构造出足以胜任 10K 以上并发连接响应的超高性能 Web 应用系统。

360，UPYUN，阿里云，新浪，腾讯网，去哪儿网，酷狗音乐等都是 OpenResty 的深度用户。

好了步骤开始：

1、安装Luagit:

yum install -y readline-devel pcre-devel openssl-devel

wget http://luajit.org/download/LuaJIT-2.0.5.tar.gz

tar -xzf LuaJIT-2.0.5.tar.gz && cd LuaJIT-2.0.5

make && make install

export LUAJIT_LIB=/usr/local/lib && export LUAJIT_INC=/usr/local/include/luajit-2.0

ln -s /usr/local/lib/libluajit-5.1.so.2 /lib64/libluajit-5.1.so.2

#一定创建此软连接，否则会报错 如果不创建符号链接，可能出现以下异常： error while loading shared libraries: libluajit-5.1.so.2: cannot open shared object file: No such file or directory

2、安装openresty:

wget https://openresty.org/download/openresty-1.11.2.2.tar.gz

tar -zxf openresty-1.11.2.2.tar.gz && cd openresty-1.11.2.2

./configure --prefix=/usr/local/openresty \ --user=www \ --group=www \ --with-luajit \ --with-http_v2_module \ --with-http_stub_status_module \ --with-http_ssl_module \ --with-http_gzip_static_module \ --with-ipv6 --with-http_sub_module \ --with-pcre \ --with-pcre-jit \ --with-file-aio \ --with-http_dav_module

gmake && gmake install

3、测试openresty:

vim /usr/local/openresty/nginx/conf/nginx.conf     可在server{..}段添加location规则



测试并启动nginx

/usr/local/openresty/nginx/sbin/nginx -t
/usr/local/openresty/nginx/sbin/nginx

测试一下访问是否输出hello world，后面应该会有一些列的简介。



4、下载开源项目：

cd /usr/local/openresty/nginx/conf/

https://github.com/bongmu/ngx_lua_waf.git

5、然后修改nginx添加配置，支持lua脚本地址，在http段位置：

lua_package_path "/usr/local/openresty/nginx/conf/ngx_lua_waf/?.lua";  ###相关项目存放地址

lua_shared_dict limit 10m;                       ###存放limit表的大小

init_by_lua_file  /usr/local/openresty/nginx/conf/ngx_lua_waf/init.lua; ###相应地址

access_by_lua_file /usr/local/openresty/nginx/conf/ngx_lua_waf/waf.lua; ##相应地址

 6、修改ngx_lua_waf相关配置：

cd ngx_lua_waf/ && vim config.lua 

RulePath = "/usr/local/openresty/nginx/conf/ngx_lua_waf/wafconf"   ##指定相应位置

attacklog = "on"                            ##开启日志

logdir = "/usr/local/openresty/nginx/conf/ngx_lua_waf/logs"           ##日志存放位置

UrlDeny="on"                              ##是否开启URL防护

Redirect="on"                             ##地址重定向

CookieMatch="on"                           ##cookie拦截

postMatch="on"                            ##post拦截

whiteModule="on"                           ##白名单

black_fileExt={"php","jsp"}                        

ipWhitelist={"127.0.0.1"}                    ##白名单IP

ipBlocklist={"1.0.0.1"}                     ##黑名单IP

CCDeny="on"                             ##开启CC防护        

CCrate="100/60"                          ##60秒内允许同一个IP访问100次

7、创建日志存放目录：

mkdir /usr/local/openresty/nginx/conf/ngx_lua_waf/logs

chown -R nobody:nobody /usr/local/openresty/nginx/conf/ngx_lua_waf/logs

8、重启nginx测试：



 

10、压力测试CC攻击：

把congfig.lua的频率改成如下：

CCDeny="on"

CCrate="50/60"

测试结果：



到处已经构建成功了一套waf防御系统，非常感谢loveshell提供这么棒的waf开源项目，还有春哥的openresty.
