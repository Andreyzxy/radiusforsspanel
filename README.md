# radiusforsspanel  


# 注意事项  
* 想要使用sspanel的radius对接，必须使用radius 2.2以内版本  
* 部署过程中可能遇到各种意想不到的问题，本文最后列出了常见问题解决办法 Q&A  
* 本实例默认你已经会使用防火墙设置，不再赘述。端口范围1812-1814
* 部署成功也不会有ss那样的在线提示  

# 软件安装  

### 服务端(centos7):  

* 先安装perl  
```  
yum install perl perl-DBI perl-CPAN -y
# 然后需要安装perl的DBI组件
perl -MCPAN -e shell
cpan>install DBI
//安装完成后退出cpan
cpan>quit
```  
①  
* 再安装其它组件  
```
yum install log4cxx tncfhh tncfhh-libs tncfhh-utils xerces-cc -y
yum install unixODBC -y
yum install postgresql-libs -y
```

```
rpm -ivh https://packetfence.org/downloads/PacketFence/CentOS7/devel/x86_64/RPMS/freeradius-2.2.10-5.2.x86_64.rpm
rpm -ivh https://packetfence.org/downloads/PacketFence/CentOS7/devel/x86_64/RPMS/freeradius-mysql-2.2.10-5.2.x86_64.rpm
rpm -ivh https://packetfence.org/downloads/PacketFence/CentOS7/devel/x86_64/RPMS/freeradius-utils-2.2.10-5.2.x86_64.rpm
```
②

### 客户端(centos7):  
```
yum install radcli -y
```

# sspanel与radius对接(服务端)  

1.创建数据库 radius 和用户 radius  
2.选择数据库 导入radius  https://github.com/stardock/Radius-install/raw/master/all.sql  
3.继续设置 radius, 编辑 /etc/raddb/sql.conf, 配置 login(用户名), password(密码), radius_db(数据库名)等字段, 找到 readclients 一行，设为 yes 并去掉注释符号#    
4.覆盖文件  
```  
wget https://github.com/stardock/Radius-install/raw/master/radiusd.conf -O /etc/raddb/radiusd.conf
wget https://github.com/stardock/Radius-install/raw/master/default -O /etc/raddb/sites-enabled/default
wget https://github.com/stardock/Radius-install/raw/master/dialup.conf -O /etc/raddb/sql/mysql/dialup.conf
wget https://github.com/stardock/Radius-install/raw/master/dictionary -O /etc/raddb/dictionary
wget https://github.com/stardock/Radius-install/raw/master/counter.conf -O /etc/raddb/sql/mysql/counter.conf
```  
5.   
测试启动  
```  
radiusd -X
```  
配置完成  
```
service radiusd start
chkconfig radiusd on
```  
③  
注意：配置freeradius可能导致mysql无法启动，此时只需要删除 `/etc/mysql/my.cnf` 即可。  

6.配置 sspanel
```
cd /www/wwwroot/你的域名
vi config/.config.php
```

```
$System_Config['enable_radius']='false'; // 配置了 radius 的话就开
#Radius数据库设置
$System_Config['radius_db_host']='';		// 跟 上面 database 数据库配置差不多 换成radius即可
$System_Config['radius_db_database']='';
$System_Config['radius_db_user']='';
$System_Config['radius_db_password']='';
#Radius连接密钥
$System_Config['radius_secret']=''; // 这个重要 必须设
```

7.Crontab添加下面这三条
```  
*/1 * * * * php /www/wwwroot/[你的网站]/xcat synclogin
*/1 * * * * php /www/wwwroot/[你的网站]/xcat syncvpn
*/1 * * * * php -n /www/wwwroot/[你的网站]/xcat syncnas
```  

8.在网站中添加节点  
首先新增 VPN/Radius基础，然后再新增 Anyconnect。也就是每个节点添加两个  
参考 https://github.com/q158073378252010/glzjin_ss-panel-v3-mod/wiki/%E9%AD%94%E6%94%B9%E7%89%88%E6%B7%BB%E5%8A%A0%E8%8A%82%E7%82%B9%E7%9A%84%E5%87%A0%E7%A7%8D%E6%96%B9%E5%BC%8F%E8%AF%B4%E6%98%8E  

# radius与anyconnect对接(客户端)  

* 设置freeradius客户端文件

修改 etc/radcli/radiusclient.conf  
改为 `authserver 服务端IP`  
改为 `acctserver 服务端IP`  
保存这份文件  
打开freeradius客户端的服务器设置  
修改 /etc/radcli/servers  
添加 `服务端IP testing123` 至最后一行，保存这份文件。  
保存这份文件。自此，我们已经设置好了freeradius客户端需要做的设置。我们接下来让anyconnect使用freeradius作为验证模块  


* 设置anyconnect

1.anyconnect安装
```  
yum install net-tools bind-utils -y
wget https://raw.githubusercontent.com/stardock/ocserv-auto/master/ocserv-auto.sh
bash ocserv-auto.sh
```  
④

2.修改ocserv配置文件 /etc/ocserv/ocserv.conf。  
将第一行的验证方式修改为：  
```
auth = "radius[config=/etc/radcli/radiusclient.conf,groupconfig=true]"
```
这样，我们就使得ocserv使用freeradius来进行用户验证。  
然后使用 `systemctl restart ocserv`在后台来启动ocserv。  


Q&A  
①DBI安装出现问题请参考 https://blog.51cto.com/lxsym/484820 清理CPAN  
②Radius的安装需要DBi支持  
③若启动失败，请使用 Radiusd -X 查看输出信息  
④Reference: https://github.com/stardock/ocserv-auto  
⑤Radius启动失败提示libssl安全风险请参考 https://serverfault.com/questions/803689/freeradius-cant-get-new-openssl-version  
`allow_vulnerable_openssl = yes`  

Thanks for refering:  
https://zorz.cc/post/install-sspanel-v3-web-control-log.html  
https://zorz.cc/post/use-ocserv-with-sspanel-v3-mod-on-debian8.html  
https://www.opscaff.com/2017/03/01/ocserv-radius-%E8%AE%A4%E8%AF%81%E6%90%AD%E5%BB%BA-freeradius-mysql/  
https://www.cnblogs.com/opsprobe/p/9769555.html  
https://www.qiyichao.cn/archives/3/  
RPM仓库: https://packetfence.org/downloads/PacketFence/CentOS7/devel/x86_64/RPMS/  
