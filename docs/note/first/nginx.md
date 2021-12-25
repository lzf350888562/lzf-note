# 安装与使用 

centos7安装nginx

yum版本安装的nginx目录为 /etc/nginx

```
yum install nginx
systemctl start nginx
```

接下来可以通过80端口(默认监听)进行访问.

查看默认页面位置

```
whereis nginx
cd /usr/share/nginx            #在该目录下
cd html/
cat "hello world" > index.html  #修改页面内容
```

再访问80可看到页面内容发送变化

## 配置

/etc/nginx/nginx.conf

可修改默认端口然后restart再访问

# 反向代理

nginx80代理tomcat8080

centos7安装tomcat 并运行  /bin/startup.sh

修改nginx/nginx.conf  (先备份一份),在http{}中

```
 server {
        listen          80;
        server_name     192.168.153.134;
    
        location / {
            root        html;
            proxy_pass  http://127.0.0.1:8080;
            index       index.html index.htm;
        }
    }
```

启动nginx

yum版本为

```
systemctl start nginx.service
systemctl stop nginx.service
```

如果出现502 gateway bad:

```
setsebool -P httpd_can_network_connect 1
```

 

如果在反向代理时候需要根据不同的路径跳转到不同的端口服务.

nginx9001分别代理tomcat8080 tomcat8081的不同路径

修改8081 tomcat 配置文件conf/server.xml  端口 .

并在8080 8081的webapps下分别放置一些html页面, 启动两个tomcat

.....

配置文件

```
 server {
        listen          9001;
        server_name     192.168.153.134;
    
        location ~ /edu/ {
            proxy_pass  http://127.0.0.1:8080;
        }
        location ~ /vod/ {
            proxy_pass  http://127.0.0.1:8081;
        }
    }
```



注意 如果uri中包含正则,则必须有~标识.

# 负载均衡

访问http://192.168.153.134/edu/a.html,负载均衡效果分担给8080 和 8081

修改配置文件nginx.conf,在http{}中 upstream与server同级

```
upstream myserver{
	server 192.168.153.134:8080;
	server 192.168.153.134:8081;
}
server {
        listen          80;
        server_name     192.168.153.134;
    
        location / {
            root        html;
            proxy_pass  http://myserver;  #指明负载均衡名字
            index       index.html index.htm;
        }
}
```

负载均衡策略:

1,轮询(默认)

2.weight(加载proxy_pass后)

代表轮询权重,默认1,权重越高被分配的客户端越多

3.ip_hash(加在upstream中)

对每个请求按访问ip的hash结果进行分配,这样每个访客固定访问一个后端服务器.

4.fair(加在upstream中,第三方)

按后端服务器的响应时间来分配请求,响应时间短的优先分配.



# 动静分离

简单理解就是nginx处理静态页面,Tomcat处理动态页面.也是基于反向代理.



假设存在资源目录/data/www 和/data/image

```
location /www/ {
	root		/data/;
	index		index.html index.ntm;
}
location /image/ {
	root		/data/;
	autoindex	on;   #访问目录时列出资源
}
```



# 高可用

虚拟ip

主master  keepalived

备master  keepalived

前提:两台服务器 192.168.17.129 192.168.17.131(vm克隆),两台服务器安装nginx和keepalived(yum安装)

修改keepalived.conf文件

```
! Configuration File for keepalived

global_defs {
    notification_email {
        acassen@firewall.loc
        failover@firewall.loc
        sysadmin@firewall.loc
     }
        notification_email_from Alexandre.Cassen@firewall.loc
        smtp_server 192.168.17.129
        smtp_connect_timeout 30
        router_id LVS_DEVEL # 主机名字
}
vrrp_script chk_http_port {
        script "/usr/local/src/nginx_check.sh"
        interval 2 #（检测脚本执行的间隔）
        weight 2 # 权重
   }
        vrrp_instance VI_1 {
        state MASTER # 备份服务器上将 MASTER 改为 BACKUP
        interface eth1 # 网卡
        virtual_router_id 51  # 主、备机的 virtual_router_id 必须相同
        priority 100  # 主、备机取不同的优先级，主机值较大，备份机值取较小90
        advert_int 1
        authentication {
                auth_type PASS
                auth_pass 1111
        }
        virtual_ipaddress {
             192.168.77.50 # VRRP H 虚拟地址
        }
}
```

router_id LVS_DEVEL  在/etc/host中添加127.0.0.1  LVS_DEVEL 映射, 可以省略,不通过映射访问.

上述指定的地方新建脚本文件"/usr/local/src/nginx_check.sh"

```
#!/bin/bash
A=`ps -C nginx –no-header | wc -l`
if [ $A -eq 0 ];then
        /etc/nginx
        sleep 2
        if [ `ps -C nginx --no-header |wc -l` -eq 0 ];then
            killall keepalived
        fi
fi
```

先启动nginx

centos7启动keepalived

```
systemctl start keepalived.service
```



测试

浏览器访问虚拟ip  192.168.77.50

在ifconfig ens33可以看到 绑定的虚拟ip.

测试可用性

停止主服务器 在访问虚拟ip, 同样可以正常访问.

在ifconfig ens33可以看到 绑定的虚拟ip.