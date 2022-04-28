# Nginx开源版使用手册

> author：chuyangc
>
> email：chu_yang1025@163.com
>
> environment：CentOS 7
>
> 在本手册中，搭建了四台虚拟机，以下是笔者搭建时机器的内网IP，请读者自行搭建环境。
>
> Nginx1@IP：192.168.244.137
> Nginx2@IP：192.168.244.140
> Nginx3@IP：192.168.244.141
> Nginx4@IP：192.168.244.142


## Nginx的基本使用

### Nginx是什么

Nginx是一个高性能的HTTP和反向代理web服务器，同时也提供了IMAP/POP3/SMTP服务。Nginx是由伊戈尔·赛索耶夫为俄罗斯访问量第二的Rambler.ru站点开发的，第一个公开版本0.1.0发布于2004年10月4日。其将源代码以类BSD许可证的形式发布，因它的稳定性、丰富的功能集、简单的配置文件和低系统资源的消耗而闻名。2011年6月1日，nginx 1.0.4发布。Nginx是一款轻量级的Web 服务器/反向代理服务器及电子邮件（IMAP/POP3）代理服务器，在BSD-like 协议下发行。其特点是占有内存少，并发能力强，事实上nginx的并发能力在同类型的网页服务器中表现较好，guo使用nginx网站用户有：百度、京东、新浪、网易、腾讯、淘宝等。

### Nginx的安装

* [Nginx的开源版本](http://nginx.org/)
* [Nginx Plus商业版本](https://www.nginx.com)
* [Openresty](https://openresty.org/cn/)
* [Tengine](http://tengine.taobao.org/)

### 通过wget获取Nginx开源版本1.21.6并安装

```shell
#获取Nginx
wget http://nginx.org/download/nginx-1.21.6.tar.gz
#解压
tar -xzvf nginx-1.21.6.tar.gz

#在安装前请确认有以下库文件
yum -y install gcc pcre pcre-devel zlib zlib-devel

#进入目录进行安装
cd nginx-1.21.6

#依次执行以下命令以安装程序，设定安装目录在/usr/local/nginx(通常软件会安装在/usr/local目录下，以方便管理)
./configure --prefix=/usr/local/nginx
make
make install

#进入安装目录/usr/local/nginx/sbin
##启动Nginx
./nginx
##快速停止
./nginx -s stop
##慢停止，等待处理好当前已经接受的连接请求再退出
./nginx -s quit
##重新加载配置
./nginx -s reload

#为方便访问index页面，在虚拟机环境下可关闭防火墙
systemctl stop firewalld
systemctl disable firewalld
#若是在企业环境里，可以编辑防火墙设置放行80端口
firewall-cmd --zone=public --add-port=80/tcp --permanent
firewall-cmd --reload

#安装Nginx系统服务
##创建服务脚本
vim /usr/lib/systemd/system/nginx.service
###脚本内容（ESC，冒号+wq）保存并退出
[Unit]
Description=nginx - web server
After=network.target remote-fs.target nss-lookup.target
[Service]
Type=forking
PIDFile=/usr/local/nginx/logs/nginx.pid
ExecStartPre=/usr/local/nginx/sbin/nginx -t -c /usr/local/nginx/conf/nginx.conf
ExecStart=/usr/local/nginx/sbin/nginx -c /usr/local/nginx/conf/nginx.conf
ExecReload=/usr/local/nginx/sbin/nginx -s reload
ExecStop=/usr/local/nginx/sbin/nginx -s stop
ExecQuit=/usr/local/nginx/sbin/nginx -s quit
PrivateTmp=true
[Install]
WantedBy=multi-user.target

##重新加载系统服务
systemctl daemon-reload

#启动Nginx服务
systemctl start nginx.service
#设置Nginx服务开机启动
systemctl enable nginx.service
```

### 目录结构与基本运行原理

* 目录

```shell
#进入Nginx目录
cd nginx-1.21.6/
#安装tree工具生成该项目的树结构
yum -y install tree
tree
```

* 更改默认页

请修改`/usr/local/nginx/html`目录下的**index.html**

* 基本运行原理

在目录/sbin/下有一个可执行文件名为nginx，在启动nginx服务后系统会创建nginx主进程master，主进程master会读取并校验`/conf/nginx.conf`文件里的配置信息，并fork()开启多个子进程worker响应用户的请求，当用户在互联网上访问http://192.168.244.137/index.html 时，由浏览器发送请求到应用服务器192.168.244.137上，子进程worker会对该请求进行处理，首先会读取配置信息文件`/conf/nginx.conf`并获取**index.html**的位置`/html/index.html`，另一个子进程worker会读取`/html/index.html`文件并响应给用户，用户便可看见**index.html**的内容，在响应完请求后，子进程worker还会存在于系统一段时间，完成所有之前用户的请求，并拒收新来的请求，当所有的任务完成后该子进程worker便会被kill杀死。综上所述，主进程master负责协调子进程worker工作，子进程worker负责响应用户请求；主进程master，多个子进程worker会一直存在于系统服务中，master进程会根据业务量不断地fork和kill子进程worker。

## Nginx配置与应用场景

### nginx.conf配置文件

* 最小配置文件

* **worker_processes**（取决于当前服务器的CPU内核数）

  * worker_processes 1; 默认为1，表示开启一个业务进程

* **worker_connections**（每个子进程worker可以创建多少个连接数）

  * worker_connections 1024; 单个业务进程可接受连接数

* **include mime.types**（由服务器端定义，标明了http协议头的文件类型，浏览器根据服务器返回的类型格式打开服务器返回的文件）

  * include mime.types; 引入http mime类型

* **default_type application/octet-stream**（默认格式）

  * default_type application/octet-stream; 如果mime类型没匹配上，默认使用二进制流的方式传输。

* **sendfile on**

  * sendfile on; 使用linux的 sendfile(socket, file, len) 高效网络传输，也就是数据0拷贝。

* **keepalive_timeout 65**（保持连接的超时时间）

  * keepalive_timeout 65

* server

  * ```shell
    server {
        listen 80; #监听端口号
        server_name localhost; #主机名、域名
        location / { #匹配路径
            root html; #文件根目录
            index index.html index.htm; #默认页名称
        }
        error_page 500 502 503 504 /50x.html; #报错编码对应页面
        location = /50x.html {
        	root html;
        }
    }
    ```

### 配置本地域名解析

1. 打开本地host文件（目录`C:/Windows/Systems/drivers/etc/hosts`）并修改

2. 在文件末尾加入

   ```shell
   192.168.244.137  chuyang1025.com
   ```

3. 通过ping命令查看是否配置成功

   ```shell
   ping chuyang1025.com
   ```

### 虚拟主机

虚拟主机其实是原本一台服务器只能对应一个站点，通过虚拟主机技术可以**虚拟化**成多个站点同时对外提供服务。

#### 端口不同

```shell
server {
        listen       80;
        server_name  localhost;
        location / {
            root   html;
            index  index.html index.htm;
        }
        error_page   500 502 503 504  /50x.html;
        location = /50x.html {
            root   html;
        }
    }
    server {
        listen       88;
        server_name  localhost;
        location / {
            root   /usr/www/;
            index  index.html index.htm;
        }
        error_page   500 502 503 504  /50x.html;
        location = /50x.html {
            root   /usr/www/;
        }
    }
```

#### servername不同

* 完整匹配(可以在同一servername中匹配多个域名)

  ```shell
  server_name ccc.chuyangc.com www.chuyangc.com;
  ```

* 通配符匹配(eg:asdf.chuyangc.com)

  ```shell
  server_name *.chuyangc.com
  ```

* 通配符结束匹配(eg:ccc.ocy)

  ```shell
  server_name ccc.*;
  ```

* 正则匹配(eg:1025.chuyangc.com)

  ```shell
  server_name ~^[0-9]+\.chuyangc\.com$;
  ```

### 域名解析

#### 域名解析是什么

域名解析就是域名映射到IP地址的过程，在用户访问**chuyangc.com**的时候，会先去本地DNS服务器上找chuyangc.com的IP是什么，如果没找到，则成为另一个用户向其他DNS服务器上查找域名对应的IP地址，如果找到了，就把IP地址放在回答报文中返回给用户。这个过程会直至找到能够回答该域名对应的IP地址的DNS服务器为止。

#### 企业项目实战技术架构

* **多用户二级域名**

  提供给用户的专属地址，如**chuyangc.shop**是一个电商网站，此时有一个用户名为cy访问自己的店铺网站，则有**cy.chuyangc.shop**，然后服务器在根据cy查询cy用户的信息并返回给页面，其他用户以此类推。

* **短网址**

  假设有一个短网址生成网站：**ocy.cn**，用户A将自己的博客网站a.space通过短网址生成网站生成简短的网址**ocy.cn/asdfgh**。这个过程是这样的，由页面提交源网址a.space到服务器，由于简短网址是唯一的，那么就可以通过<u>UUID</u>来生成唯一值，而且由于**a.space**网站的IP为<u>102.100.10.25</u>，那么就会有一对键值对，key是**UUID**，value是**102.100.10.25**，那么其它用户访问用户A的博客网站的短网址的时候，服务器会把博客真实的地址<u>102.100.10.25</u>返回给浏览器，并且进行重定向网页，以此来实现短网址

* **httpdns**

  httpdns是存在于**应用层**，使用者通常还是APP移动端应用或者是传统的C/S架构，httpdns技术会预先把nginx服务器的IP地址写在应用上，当应用访问页面时，便会使用预先写好的IP的地址来发起请求。

## 反向代理

**在看反向代理前，先了解一下什么是代理和正向代理。**

### 代理

代理就是客户给代理人提出一个需求，由代理人给客户去解决这个需求。

### 正向代理

假设存在这样一个场景，学生A是借呗（<u>借呗</u>现名信用贷是支付宝推出的一款贷款服务）的用户，学生B并没开通借呗。在学生B月尾的时候，因为上中旬娱乐话费开销太大，B一时没有控制住自己的欲望，面临月底吃土的情况。因为学生A和学生B是好哥们，学生B来找学生A借钱以求度过艰难的月底生活，但是学生A这个月也刚好在王者荣耀里买了个特效皮肤，所剩的零钱刚刚好够他自己度过月底，这时学生B来找自己借钱，学生A是个仗义的人，不忍看着好兄弟吃土，便一口答应要帮学生B。接着，学生A便向借呗借钱，然后再把借来的钱转账给好兄弟学生B，学生B安稳地度过了这个月底。

在这个过程中，学生B向学生A借钱，学生A向借呗借钱，学生B没有开通借呗而学生A开通了借呗且拥有相应的额度，学生A把从借呗借来的钱又借给了学生B，借呗并不知道它给学生A的钱其实是学生B在用，借呗只知道它把钱借给了学生A，但是不知道实际使用的是学生B。这便是正向代理，client要实现某个功能，但是自身无法实现，要借助serverA提交请求给serverB，由serverB来实现client想实现的功能，serverB服务于serverA，serverA服务于clinet，但是serverB不知道自己做的一切是为了client，也就是说屏蔽的服务器，它不知道它的真实用户是谁。以此类推，我们常常说的VPN（Virtual Private Network，虚拟专用网络），它其实就是正向代理。

### 反向代理

**有了前面的正向代理的铺垫后，是不是就大概能猜出来什么是反向代理了？**

那么反向代理就是屏蔽了客户（client），由代理agent来**分配**客户的请求到服务器（server）上。在这个过程中，客户并<u>不知道</u>自己的请求由哪个服务器来响应。

### 隧道式代理

综上所述，反向代理就是client发请求到代理server，再由代理server把请求转发到应用server这么一个过程。但是，无论是正向代理还是反向代理，都需要一个**代理的server**。在高并发的环境下，响应客户请求的快慢就取决于中转请求的那台server服务器的性能，而不是应用server。假如代理server的网络是百兆，而应用server的网络是千兆，那么最终服务器的网络还是百兆，不但造成了应用服务器资源浪费，也增加了代理服务器的故障率。每个客户端发出的请求，都必须要经过代理服务器（Nginx）再到应用服务器，再由应用服务器响应数据给回代理服务器，中转服务器再把数据返回给客户端，这便是<u>隧道式</u>代理。

### LVS

为了解决隧道式代理的瓶颈，就有了**DR**模型，这是**LVS**所提供的技术，LVS是一个<u>更高性能</u>的负载均衡器，但是功能却没有Nginx那么多。

#### DR模型

在客户端发出请求的时候，请求会先到代理服务器，再由代理服务器把请求转发给应用服务器，应用服务器响应请求返回数据的时候，不会再把信息返回给代理服务器，而是直接返回给客户端。也就是说，请求的时候经过了代理服务器，但是返回数据的时候不经过代理服务器。

### 轮询

在默认情况下代理服务器会使用轮询方式，逐一向应用服务器转发，这种方式适用于无状态请求。

### 会话

**会话**其实是开发者为了实现中断和继续等操作，把客户端和服务器之间的一对一交互抽象出来的概念，

#### 有状态会话(Stateful)

在客户访问某个网站的时候，浏览器会向服务器发出请求，服务器会创建一个由当前客户独享的**session对象**，完成以后会把**session id**和客户需要的数据一并返回给浏览器，session id<u>会存到浏览器临时缓存目录下的cookies文件</u>，由于服务端的session依赖于session id，并且是要根据session id找到这个客户相应的信息，也就是说在下次该用户访问该网站时候，会把cookies和请求一并传递到服务器端，在客户离开网站后服务端就会把相应的session对象销毁掉，所以用户在访问同一网站下的网页时就不需要再键入信息，这就保持了一个会话，但是session也有缺陷，就是假如服务器做了负载均衡，那么下一个操作请求到了另一台服务器的时候session对象就会丢失。

#### 无状态会话(Stateless)

生成和销毁会话都**不会保存信息**。在无状态会话下，把用户信息通过算法生成一个字符串，也就是token，在首次登录进网站的时候，服务端会把生成的token和数据一并返回给客户端，客户端在下次请求该网站的时候会在请求头上加入token，服务端收到后将token进行解密，然后获取用户信息，也就是说token 不需要再存储用户信息，节约了内存。token 虽然很好的解决了 Session 的问题，但仍然不够完美。服务端在认证 token 的时候，仍然需要去数据库查询认证信息做校验。

## 基于反向代理的负载均衡

用户发出请求到代理服务器，由代理服务器把请求转发给serverA，假如serverA出现故障了，就会把请求转发给serverB，如果serverB也不行了，那就把请求转发给serverC，这便是负载均衡。每次转发请求的时候，代理服务器都会**轮询**所有可使用的server，如果某一台server宕机了，那么就会将它移下线，再找一台server来处理请求。在轮询的过程中，也会是设定一个超时值，超过了请求响应时间，就向另一台server发出请求。

### 配置反向代理

* 首先准备三台虚拟机，Nginx1、Nginx2和Nginx3

* 然后分别修改Nginx目录下的html目录里的**index.html**文件并标识

  ```shell
  #Nginx1  /usr/local/nginx/html/index.html
  vim index.html
  #按i进入编辑模式
  here is 192.168.244.137
  #ESC 冒号+wq保存退出
  
  #Nginx2  /usr/local/nginx/html/index.html
  vim index.html
  #按i进入编辑模式
  here is 192.168.244.140
  #ESC 冒号+wq保存退出
  
  #Nginx3  /usr/local/nginx/html/index.html
  vim index.html
  #按i进入编辑模式
  here is 192.168.244.141
  #ESC 冒号+wq保存退出
  ```

* 配置nginx.conf，并且重启nginx服务

  ```shell
  #Nginx1  /usr/local/nginx/conf/nginx.conf
  vim nginx.conf
  #按i进入编辑模式，修改server下的配置信息
      server {
          listen       88;
          server_name  localhost;
          location / {
          #让请求转发到Nginx2机子上
          	proxy_pass 192.168.244.140;
          #配置proxy_pass后以下配置失效
              #root   /usr/www/;
              #index  index.html index.htm;
          }
          error_page   500 502 503 504  /50x.html;
          location = /50x.html {
              root   /usr/www/;
          }
  #ESC 冒号+wq保存退出
  
  #重启nginx服务
  systemctl reload nginx
  
  #测试是否成功
  curl 192.168.244.140
  ```

### 配置负载均衡

```shell
#此处使用Nginx1作为负载均衡服务器
##Nginx1  /usr/local/nginx/conf/nginx.conf
vim nginx.conf
###按i进入编辑模式，修改server下的配置信息
    upstream https {
    #设置转发的服务器组
	server 192.168.244.140:80;#Nginx2
	server 192.168.244.141:80;#Nginx3
    }

    server {
        listen       80;
        server_name  localhost;
        location / {
		proxy_pass	http://https;#https是别名
		#配置proxy_pass后以下配置失效
            #root   html;
            #index  index.html index.htm;
     }
####注意别名upstream要与server同级

#测试，多次测试可看见效果
curl 192.168.244.137
```

### 负载均衡策略

1. 使用权重值

   ```shell
   #现在有Nginx1、Nginx2、Nginx3、Nginx4四台机器，其中Nginx1作为代理服务器
   ##要分配权重值可在Nginx1的nginx.conf中配置upstream的weight值
   ###权重用于服务器性能不均的情况，根据权重值设定服务器被轮询到的几率
   ####eg
       upstream https {
       #设置转发的服务器组
   	server 192.168.244.140:80 weight=5;#Nginx2会被多次分配到请求
   	server 192.168.244.141:80 weight=4;#Nginx3其次
   	server 192.168.244.142:80 weight=1;#Nginx4最少
       }
   ```

2. 使用down属性让机器不再接收请求，使其下线

   ```shell
   #现在有Nginx1、Nginx2、Nginx3、Nginx4四台机器，其中Nginx1作为代理服务器
   ##要分配权重值可在Nginx1的nginx.conf中配置upstream的weight值
   ###eg
       upstream https {
       #设置转发的服务器组
   	server 192.168.244.140:80 weight=5 down;#Nginx2会下线
   	server 192.168.244.141:80 weight=4;#Nginx3现在会接收到大部分的请求
   	server 192.168.244.142:80 weight=1;#Nginx4接收少部分请求
       }
   ```

   

3. 使用backup属性使机器成为备用服务器

   ```shell
   #在其它机器宕机或者繁忙的时候，会把请求转发给带有backup属性的机器
   #eg
       upstream https {
       #设置转发的服务器组
   	server 192.168.244.140:80 weight=5 down;#Nginx2会下线
   	server 192.168.244.141:80 weight=4 backup;#Nginx3现在会接收请求
       }
   ```

### 动静分离（适用于中小型网站）

#### 使用场景

一般来说，我们打开一个网站，它不仅仅是返回了<u>页面信息</u>，还有许多**静态资源**文件如：js和css等，而服务器是用作处理动态请求的，对于这些静态资源可以把它们打包放在代理服务器上，**把动态请求和静态请求分离开**。那么下次进入网站的时候，浏览器就会从代理服务器上获取静态资源文件而不用把静态请求转发给服务器，然后把动态请求转发到服务器上。

#### 动静分离配置

```shell
#先安装Tomcat8.5，笔者使用Nginx4来运行Tomcat
##使用wget获取安装包
wget https://dlcdn.apache.org/tomcat/tomcat-8/v8.5.78/bin/apache-tomcat-8.5.78.tar.gz

##使用tar解压
tar -xzvf apache-tomcat-8.5.78.tar.gz

##移动解压后的文件夹到目录/usr/local/Tomcat8.5上
mv apache-tomcat-8.5.78 /usr/local/Tomcat8.5

##切换到安装目录，给bin/目录下的所有.sh文件授权
cd /usr/local/Tomcat8.5/
cd bin/
chmod +x *.sh

##启动Tomcat
./startup.sh

#将Nginx1作为代理服务器，配置其/usr/local/nginx/conf/目录下的nginx.conf
#下面开始配置简单的动静分离
vim nginx.conf
##写入以下配置
###转发请求到本地的8080端口，访问Tomcat
location / {
    proxy_pass http://127.0.0.1:8080;
    root html;
    index index.html index.htm;
}
####添加几个location（与上面的location同级）静态资源的配置
#####浏览器会从代理服务器中的目录/usr/local/nginx/static下分别获得css文件、images文件和js文件
######普通（非正则）location会一直往下，直到找到匹配度最高的（最大前缀匹配），以下匹配规则优先级比上面的location高
location /css {
    root /usr/local/nginx/static;
    index index.html index.htm;
}
location /images {
    root /usr/local/nginx/static;
    index index.html index.htm;
}
location /js {
    root /usr/local/nginx/static;
    index index.html index.htm;
}
```

##### 也可以使用简单的正则表达式匹配

```shell
location ~*/(css|img|js) {
    root /usr/local/nginx/static;
    index index.html index.htm;
}
```

### URLRewirte

#### 语法以及参数

```shell
rewrite是实现URL重写的关键指令，根据regex (正则表达式)部分内容，
重定向到replacement，结尾是flag标记。
#一般是配置在location或者是在server上，加入以下语句

 rewrite <regex> <replacement> [flag];
-关键字    正则     替代内容     flag标记

#其中
    关键字：其中关键字error_log不能改变
    正则：perl兼容正则表达式语句进行规则匹配
    替代内容：将正则匹配的内容替换成replacement
    flag标记：rewrite支持的flag标记
    
flag标记说明：
    last #本条规则匹配完成后，继续向下匹配新的location URI规则
    break #本条规则匹配完成即终止，不再匹配后面的任何规则
    redirect #返回302临时重定向，浏览器地址栏会显示跳转后的URL地址
    permanent #返回301永久重定向，浏览器地址栏会显示跳转后的URL地址
```

#### 示例:

```shell
#使用10.html替换了原来的/index.jsp?arg=10
rewrite ^/([0-9]+).html$ /index.jsp?arg=$1 break;
#用户看见的网址是：http://IP/10.html
```

#### 与反向代理和负载均衡一起使用

##### 代理服务器？网关服务器？

在代理服务器上同时使用URLRewrite、反向代理和负载均衡的时候，它就不仅仅是提供了代理的功能了，这时候管它叫<u>网关服务器</u>较为准确。

##### 网关服务器

在实际生产环境中，应用服务器是不能被用户直接访问到的，都要经过网关服务器，**通过网关服务器转发请求到应用服务器，再由应用服务器响应请求**，在这个过程中，网关服务器就好比餐厅的门，应用服务器就好比餐桌，用户要通过这扇门，才能进去到里面的餐桌上就餐。

#### 配置步骤

1. 开启应用服务器上的防火墙服务（相关命令见附录）

2. 配置指定的IP和端口访问

3. 配置网关服务器的**nginx.conf**文件

    _TODO_  \<regex>、\<replacement>、\<arguments>

```shell
#nginx.conf
    upstream httpds {
        server 192.168.244.140:8080 weight=8 down;
        server 192.168.244.141:8080 weight=2;
        server 192.168.244.142:8080 weight=1 backup;
    }
    location / {
        rewrite <regex>$ /<replacement>?<arguments>=$1 redirect;
        proxy_pass http://httpds ;
    }
```

### 配置防盗链

#### 盗链

浏览器在加载一个站点的首页的时候（假定是**index.html**），会读取里面的img和script等标签，在这些标签内一般会有src属性，它是一个绝对的URL地址或者是该服务器的相对路径地址，然后就对静态资源下的src的链接发送GET请求获得资源，各种资源会渲染整个页面，最后浏览器根据html根据语法进行格式排版并呈现一个最终的页面。

**假定存在一种场景，分别有两个网站，a.com、b.com。如果有大量的客户访问a站点的时候，把静态资源的地址填上了b站点的静态资源地址，那么就会出现访问a站点却是消耗b站点的流量的情况，在a站点页面资源渲染的时候获取了非本站的静态资源文件，这便是盗链。**

网页是会发出很多个请求去获取资源的，在<u>重复访问</u>一个网站的时候，浏览器会给请求头加上一个参数**referer**（可自行打开浏览器开启开发者模式测试，打开network标签并观察每一个资源的加载过程），这是由http协议约定并由浏览器执行的，它的参数显示<u>引用页</u>。

#### 配置

```shell
valid_referers [none | blocked | server_names] Your IP/Your Domain;
#none， 检测Referer头域不存在的情况。
#blocked，检测Referer头域的值被防火墙或者代理服务器删除或伪装的情况。这种情况该头域的值不以“http://” 或 “https://”开头。
#server_names,设置一个或多个 URL，检测 Referer 头域的值是否是这些 URL 中的某一个。
```

在需要配置防盗链的location下

_TODO_  <Your IP/Your Domain>

```shell
valid_referers <Your IP/Your Domain>;#配置允许引用资源的IP或者是域名
if ($invalid_referer) {				#如果不是允许引用的地址则返回状态码403
	return 403;
}
```

#### 测试

使用curl进行测试

```shell
#显示请求头部信息
curl -I <IP/Domain>

#显示http请求的通信过程
curl -v <IP/Domain>

#指定来源地址为your.com
curl -e "your.com" http://www.baidu.com -v
```

#### 返回错误页面（自行准备403.html并放在html目录下）

_TODO_  <Your IP/Your Domain>

```shell
valid_referers <Your IP/Your Domain>;#配置允许引用资源的IP或者是域名
    if ($invalid_referer) {				#如果不是允许引用的地址则返回状态码403
        return 403;
}

	location /403.html {
        root html;
    }
```

#### 使用URLRewrite配置返回的可视化错误信息（自行准备403.png）

```shell
#TODO：<Your IP/Your Domain>
valid_referers <Your IP/Your Domain>;#配置允许引用资源的IP或者是域名
if ($invalid_referer) {				#如果不是允许引用的地址则返回403.png
	rewrite ^/ /img/403.png;		#匹配所有目标并匹配到img目录下的403.png
}
```

## 高可用（HA，High Availability）

### 高可用

首先来聊一下什么是高可用，高可用是分布式系统架构设计中必须要考虑的因素之一，通过设计以达到减少系统不能提供服务的时间的目的。在单位时间里（通常是一年），服务器能够工作的时间比例。那么，如何来衡量系统是否高可用呢？假设，现在有一个系统X，X在一年里都是正常提供服务，这时候可以说X的可用性是100%，当然了，这种情况也只是在理想的状态下，在实际的生产应用环境下，服务器的可用性一般是以多少个9来表示，例如：99%、99.999%，9越多就代表可用性越强，计算公式为：` 可用性 = 平均故障间隔 / (平均故障间隔 + 故障恢复的平均时间) `

目前大多数的企业高可用的目标是4个9，也就是99.99%，在这台服务器运行一年的时间内停机的时间约为53分钟。在国内，公认的高可用网站大家都能想到是baidu.com，人们甚至会通过baidu.com能不能访问来判断网络的连通性，百度的高可用服务给大家留下了“网络畅通，百度就能访问”的印象。

### 配置

```shell
#首先需要安装Keepalived
##下载地址：https://www.keepalived.org/download.html，需要安装openssl-devel依赖
###这里需要完整克隆一下Nginx虚拟机，然后进行以下操作

#1.本文使用yum安装Keepalived
yum -y install keepalived

#2.编辑Keepalived配置文件/etc/keepalived/keepalived.conf

#3.启动Keepalived服务
systemctl start keepalived
```

第一台机器的配置（IP：192.168.244.140）

```shell
! Configuration File for keepalived

global_defs {
   router_id 140 		#id随意
}

vrrp_instance 140 {		#名称随意
    state MASTER
    interface ens33		#编写网卡设备名称
    virtual_router_id 51
    priority 100		#优先级，优先级越高的那台机子就是Master
    advert_int 1		#间隔检测的时间
    authentication {	#认证，在跑同一组Keepalived服务时需要配置一致
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
        192.168.244.200 #虚拟IP
    }
}
}
```



第二台机器的配置（IP：192.168.244.143）

```shell
! Configuration File for keepalived

global_defs {
   router_id 143 		#id随意
}

vrrp_instance 143 {
    state MASTER
    interface ens33		#编写网卡设备名称
    virtual_router_id 51
    priority 100		#优先级，优先级越高的那台机子就是Master
    advert_int 1		#间隔检测的时间
    authentication {	#认证，在跑同一组Keepalived服务时需要配置一致
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
        192.168.244.200	#虚拟IP
    }
}
}
```

### 测试

在宿主机中使用ping命令

```shell
ping 192.168.244.200
```

- 此时可以将Nginx1进行关机，观察ping命令的返回结果

### 结论

Keepalived是**主机/进程级别**的监控，可以判断当前运行Keepalived进程的机子是否宕机，由于现在Nginx1和Nginx1 backup都使用了**虚拟IP**（VIP，Vrtual IP Address）192.168.244.200，那么就是说，访问这个IP地址就是访问这两台机子，如果宕机了，keepalived会自动识别到宕机的机子然后把请求发给另一台机子。虽然Keepalived进程可以检测主机是否宕机，但是却无法检测Nginx服务是否正常运行，<u>假如有一台机子的Nginx服务停掉了，Keepalived是检测不到的。</u>对于这个问题的解决方案呢，可以使用脚本检测Nginx服务，然后使用Keepalived检测主机是否宕机，如果Nginx服务停止掉了，脚本就把Keepalived进程杀死。

> **本文仅代表个人观点，用于学习与交流。感谢你能看到这里，希望对你有所帮助！**

# 附录

1. yum更换国内镜像源

   ```shell
   #1、备份
   mv /etc/yum.repos.d/CentOS-Base.repo /etc/yum.repos.d/CentOS-Base.repo.backup
   #2、下载到etc/yum.repos.d
   wget -O /etc/yum.repos.d/CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-7.repo
   ```

2. 运行出现："Existing lock /var/run/yum.pid: another copy is running as pid..."

   ```shell
   rm -f /var/run/yum.pid
   /sbin/service yum-updatesd restart
   ```

3. CentOS防火墙服务相关命令

   _TODO_ \<Your IP>、<tcp/udp>、\<Your port>
   
   ```shell
   #开启防火墙
   systemctl start firewalld
   
   #重启防火墙
   systemctl restart firewalld
   
   #重新加载配置
   firewall-cmd --reload
   
   #查看现有配置
   firewall-cmd --list-all
   
   #指定指定IP和端口访问
   firewall-cmd --permanent --add-rich-rule="rule family="ipv4" source address="<Your IP>"
   port protocol="<tcp/udp>" port="<Your port>" accept"
   
   #移除指定IP和端口访问
   firewall-cmd --permanent --remove-rich-rule="rule family="ipv4" source
   address="<Your IP>" protocol="<tcp/udp>" port port="<Your port>" accept"
   ```

   



