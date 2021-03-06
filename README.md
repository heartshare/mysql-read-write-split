# mysql主从复制及读写分离环境的搭建(基于二进制日志文件)
环境:ubuntu 18.04(虚拟机),mysql5.7


主要步骤:  
<a href="#step1">1. 配置mysql主库</a>  
<a href="#step2">2. 配置mysql从库</a>  
<a href="#step3">3. 在主库中创建用户供从库用于复制</a>  
<a href="#step5">4. 备份主库数据,恢复到从库中</a>  
<a href="#step6">5. 在从库中配置主库的相关信息</a>  
<a href="#step7">6. 测试主从复制</a>  
<a href="#step8">7. 使用MaxScale配置读写分离</a>  
<a href="#step9">8. 使用MaxScale配置读写分离</a>  
<a href="#step10">9. 测试读写分离</a>  
<a href="#step11">10. 发现的问题</a>  
<a href="#step10">11. 参考资料</a>  


## <span id=step1>配置mysql主库</span>
- 启用二进制日志功能,在my.cnf文件,加入如下配置:
``` bash
log_bin = /var/log/mysql/mysql-bin.log 
server-id = 10
```
`log_bin`为二进制日志路径,需确保对该文件具有读写权限,`server-id`为主库实例id,配置`log_bin`后必须要配置server-id,否则mysql会无法启动,`server-id`取值为(1 ~ (2³²)−1),这个值要有唯一性(不能与主从复制结构中的其他mysql实例重复).


对于InnoDB引擎,加入以下两个选项,可保证主从复制时的持久性和一致性.
``` sql
innodb_flush_log_at_trx_commit = 1
sync_binlog = 1
```
- 确保主库的`skip_networking`选项为关闭状态,否则从库将无法与主库通信
可在主库中查看该值的设置:
``` sql
show variables like '%skip_networking%';
``` 
若为`ON`,需要在my.cnf中将其设置为`OFF`
- 重启mysql服务
``` sql
sudo systemctrl restart mysql
```

## <span id=step2>配置mysql从库</span>
对于从库,只需要在`my.cnf`文件里面配置`server-id`就可以了,其他设置是可选的.  
官方建议从库开启`log_bin`,二进制日志可用于数据备份以及从mysql故障中恢复数据.  

## <span id=step3>在主库中创建用户提供给从库用于复制</span>
主库必须提供给从库一个具有`replication slave`权限的账号给从库用于复制,多个从库可使用同一个账号复制,也可为每一个从库单独创建用户
``` sql
create user 'slave'@'%' identified by 'mypassword';
grant replication slave on *.* to 'slave'@'%';
flush privileges;
```

## <span id=step4>备份主库数据,恢复到从库中</span> 
在从库中使用`mysqldump`备份主库数据:  
``` bash
#--master-data选项会在备份时给数据库加锁
#同时,在备份生成的dbname.sql中包含主库的二进制日志坐标信息
mysqldump -uroot -p -h192.168.199.157 --master-data dbname > dbname.sql #将192.168.199.157替换为主库ip
```

在从库中恢复主库的数据:  
``` bash
mysql -uroot -p dbname < dbname.sql
```

## <span id=step5>在从库中配置主库的连接信息</span>  
在从库中配置主库连接信息:
``` sql
#在从库中执行
change master to 
master_host='master hostname or ip address', -- 替换为主库的域名或ip
master_user='replication_user_name', -- 替换为主库中创建的具有replication slave的账号
master_password='replication_password'; -- 替换为上述账号的密码
-- 使用带--master-data选项的mysqldump命令备份的数据库文件中已经包含了master_log_file和master_log_pos这两项配置的信息,
-- 从库恢复主库的数据时,已经修改了这两个配置的值,这里可以不用重复设置
-- master_log_file='recorded_log_file_name', -- 主库的二进制日志坐标信息
-- master_log_pos=record_log_postion; -- 主库的二进制日志坐标信息
```
完成后,从库中执行以下sql,开始主从复制:  
``` sql
start slave
``` 
使用以下命令查看从库的复制状态
``` sql
show slave status
```
若配置有误,`show slave status`会显示相关的错误信息.

## <span id=step6>测试主从复制</span>
登录主库,更新任意一个表的某个字段,如:
``` sql
update client set name = '测试主从复制2019-03-22 18:25:37' where id = 1;
```
登录从库,查询在主库中更新的表:
``` sql
select name from client where id = 1;
```
从库中`client.name`的值与主库中`client.name`的值一致则主从复制配置成功.

## <span id=step7>使用MaxScale配置读写分离</span>
- 从官网下载MaxScale(https://mariadb.com/downloads/#mariadb_platform-mariadb_maxscale),使用dpkg安装
- 安装完成后,需要在主库中为maxscale创建数据库用户
``` sql
CREATE USER 'maxscale'@'%' IDENTIFIED BY 'maxscale_pw';
GRANT SELECT ON mysql.user TO 'maxscale'@'%';
GRANT SELECT ON mysql.db TO 'maxscale'@'%';
GRANT SELECT ON mysql.tables_priv TO 'maxscale'@'%';
GRANT SELECT ON mysql.roles_mapping TO 'maxscale'@'%';
GRANT SHOW DATABASES ON *.* TO 'maxscale'@'%';
```
- 编辑MaxScale配置文件(默认位于`/etc/maxscale.cnf`中)
  `maxscale.cnf`文件中已经有了一些初始化的配置,将里面的数据库配置改为前面步骤配置的主从结构即可.
- 启动maxscale:
``` bash
sudo systemctl start maxscale #systemd
#或者
sudo service maxscale start #initd
#查看启动日志
tail -f /var/log/maxscale/maxscale.log
```
若启动失败,可通过```systemctl status maxscale```或中日志文件中查看错误信息.

## <span id=step8>使用kingshard配置读写分离</span>
除了MaxScale,还可以使用kingshard配置读写分离.  
kingshard的GitHub主页有比较详细的安装步骤:  
(https://github.com/flike/kingshard/blob/master/doc/KingDoc/kingshard_install_document.md)
以及配置文件的说明:  
(https://github.com/flike/kingshard/blob/master/doc/KingDoc/how_to_use_kingshard.md)  
在kingshard的bin目录中启动kingshard,通过`-config`选项指定配置文件路径,例:
``` bash
/usr/local/go/src/github.com/flike/kingshard/bin/kingshard -config=/etc/kingshard.yaml
```

## <span id=step9>测试读写分离</span>
若读写分离配置正确,则所有的写操作都被转发到主库中,所有的读操作都被转发到从库中.  
为了测试MaxScale或kingshard的读写分离配置正确,需在主库和从库中启用`general_log`选项,启用该选项可查询数据库中实时的sql语句.
``` sql
-- 查看是否启用general_log,以及general_log日志文件路径
show variables like '%general%'
-- 若未启用,需要将该选项置为ON
set global general_log=ON
```


查看sql语句输出:
``` bash
sudo tail -f /var/lib/mysql/xxxx.log
```

使用mysql连接到MaxScale或kingshard的mysql代理服务器上,分别执行`update`和`select`语句.  
若update语句出现在主库的log中,select语句出现在从库的log中,则读写分离配置成功.  


测试完毕,关闭`general_log`选项:
``` sql
set global general_log=OFF
```


## <span id=step10>发现的问题</span>
mysql8.0的命令行以及8.0版本的MySQLWorkbench无法连接到MaxScale和kingshard配置好的代理服务上(Mac环境,其他环境未测试).  
应用程序中,jdbc驱动`5.1.x`和`8.0.x`版本均可正常连接.

## <span id=step11>参考资料</span>
- [Setting Up Binary Log File Position Based Replication](https://dev.mysql.com/doc/refman/8.0/en/replication-howto.html)
- [Setting up MariaDB MaxScale](https://github.com/mariadb-corporation/MaxScale/blob/2.2/Documentation/Tutorials/MaxScale-Tutorial.md)
- [Read/Write Splitting with MariaDB MaxScale](https://github.com/mariadb-corporation/MaxScale/blob/2.2/Documentation/Tutorials/Read-Write-Splitting-Tutorial.md)
- [kingshard简介](https://github.com/flike/kingshard/blob/master/README_ZH.md)

