# MySQL数据存储路径迁移详细记录
 * Created by CYZ
 * DateTime: 2017-10-25
----

## 说明
本文档记录了在Ubuntu16.04系统下更改MySQL数据目录的过程，在网络上也能搜到其他人教程，但是很多都是复制的别人的，没有经过亲身的操作就PO出来。本文档经过自己实际操作，记录每一步的详细步骤。

## 材料
#### Ubuntu系统
- 系统版本：16.04 桌面版
- 系统类型：64位
#### MySQL服务端
- 软件版本：5.7.18-1
- 安装方式：通过apt安装

## 操作背景（吐槽阿里云）
由于阿里云的系统盘空间不够，需要将MySQL的数据存储到新买的数据盘上，坑就是从这里开始的。
#### 阿里云ECS不支持MySQL的分区功能
数据盘是挂载到/mnt下的，所以希望通过数据表分区的方式把文件转移到/mnt/mysql下，建表语句如下：
```
CREATE TABLE `data_mnt` (
  `id` INT (11) UNSIGNED NOT NULL AUTO_INCREMENT,
  `line_id` INT (11) UNSIGNED NOT NULL,
  PRIMARY KEY (`id`)
) ENGINE = MyISAM AUTO_INCREMENT = 1 DEFAULT CHARSET = utf8 PARTITION BY RANGE (`id`)(
  PARTITION part01
  VALUES
    LESS THAN MAXVALUE DATA DIRECTORY = "/mnt/mysql" INDEX DIRECTORY = "/mnt/mysql"
);
```
这个在本地的ubuntu下是没有问题的，但是在阿里云ECS上却得到`Permission denied`的提示：
```shell
mysql> CREATE TABLE `data_mnt` (
    -> `id` INT (11) UNSIGNED NOT NULL AUTO_INCREMENT,
    -> `line_id` INT (11) UNSIGNED NOT NULL,
    -> PRIMARY KEY (`id`)
    -> ) ENGINE = MyISAM AUTO_INCREMENT = 1 DEFAULT CHARSET = utf8 PARTITION BY RANGE (`id`)(
    -> PARTITION part01
    -> VALUES
    -> LESS THAN MAXVALUE DATA DIRECTORY = "/mnt/mysql" INDEX DIRECTORY = "/mnt/mysql"
    -> );
ERROR 1 (HY000): Can't create/write to file '/mnt/mysql/data_mnt#P#part01.MYI' (Errcode: 13 - Permission denied)
mysql> system ls -l /mnt
total 20
drwx------ 2 root  root  16384 Aug 21 16:16 lost+found
drwxrwxrwx 2 mysql mysql  4096 Oct 24 13:27 mysql
```
而实际上/mnt/mysql的权限是777，而且所有者是`mysql`。这个问题阿里云的工程师也没能解决，他在另一个服务器（CentOS）测试仍然得到一样的结果，最后不了了之。

### 软链接方式行不通
不支持分区就算了，我还想到另一个办法，这个方法在本地测试仍然没有问题，但在阿里云还是碰壁了。
我先用普通的方式先创建一个表，建表语句：
```
CREATE TABLE `data_mnt` (
  `id` int(11) unsigned NOT NULL AUTO_INCREMENT,
  `line_id` int(11) unsigned NOT NULL,
  PRIMARY KEY (`id`)
) ENGINE=MyISAM DEFAULT CHARSET=utf8;
```
嗯，这个是没有问题的。然后将/var/lib/mysql/alitest下的三个文件
```
data_mnt.frm
data_mnt.MYD
data_mnt.MYI
```
移动到/mnt/mysql下，最后将移动的文件软链接到原来的位置，更改一波权限。发现貌似可以用，我向数据表插入了1GB的数据，确实插入成功了。然而，发现文件并没有变大，数据盘空间没有减少，减少的是系统盘空间。更奇怪的是，系统盘是无缘无故减少了，找不到文件在哪里。
在本地的测试结果是，移动的文件增大了，没有什么异常。
以上2个操作均以失败告终，无奈只得迁移MySQL的数据目录，网上的教程多半坑，所以自己记录一下。

## 转移MySQL数据目录的步骤
- 停止MySQL服务
打开终端切换到root用户，输入以下命令停止MySQL
```shell
  service mysql stop
```
- 移动MySQL的数据目录
确认MySQL停止后，切换目录到/var/lib下，将/var/lib/mysql复制（或移动）到/mnt
```shell
  cd /var/lib
  cp -a ./mysql /mnt/
```
- 修改MySQL的配置文件
网络上很多教程都是复制了别人的，把数据库版本号改一下就随意贴上来，很不负责任。其中就有些瞎改的，在5.7的版本中，根本就没有/etc/my.cnf文件，有人还是说修改这个，甚至有人说如果没有要新建！！这些都不靠谱，经过我自己的探索（去查看可能与配置相关的文本文件），只需修改一处。
将路径切换到/etc/mysql/mysql.conf.d，打开mysqld.cnf，进行修改：
```shell
  cd /etc/mysql/mysql.conf.d
  vi mysqld.cnf
```
将
```shell
  datadir = /var/lib/mysql
```
改为
```shell
  datadir = /mnt/mysql
```
然后重启MySQL，大功告成！如果成功可删除原来的文件，失败就是哪里失误了，改回去重新再来。
```shell
  service mysql start
```

## 关于网上流传的一些方式的纠正（仅针对5.7）
- datadir需要修改多出
实际上不需要，有很多把低版本的教程改掉版本号就当5.7的版本发出来，datadir只需要修改/etc/mysql/mysql.conf.d/mysqld.cnf一处。
- 除了datadir，还要修改socket
也不需要，我看了他们写的大概是要把/var/lib/mysql/mysql.sock改一下，然而5.7版本的mysql.sock不在/var/lib/mysql下，通过修改的文件可以看到socket文件是/var/run/mysqld/mysqld.sock，与MySQL数据目录不在一起，因而不需要修改。


## 关于阿里云更改MySQL配置后无法启动的补充
很遗憾，转移数据库目录后在阿里云ECS无法启动MySQL服务，尽管这些操作已经在本地经过严格的测试。但是上述步骤仍可作为更改MySQL5.7数据目录的一般教程。既然这个方法在阿里云上行不通，那就只能出绝招了。
### 更改数据盘的挂载路径
目前的数据盘挂载到/mnt下，我们需要将其重新挂载到MySQL的数据目录下，即/var/lib/mysql
- 停止MySQL服务
```shell
  service mysql stop
```
- 重命名MySQL数据目录
```shell
  cd /var/lib
  mv mysql mysql.bak
```
- 数据盘重新挂载
```shell
  # 卸载数据盘
  umount /dev/vdb1
  # 格式化数据盘（可选）
  mkfs.ext4 /dev/vdb1
  # 创建/var/lib/mysql目录
  cd /var/lib
  mkdir mysql
  # 挂载数据盘
  mount /dev/vdb1 /var/lib/mysql
```
- 复制（移动）数据
```shell
  cp -a /var/lib/mysql.bak/*  /var/lib/mysql/
  # 更改MySQL所有者
  chown mysql:mysql /var/lib/mysql
```
- 启动MySQL服务
```shell
  service mysql start
```
- 确认正常运行后，删除原数据库文件
```shell
  rm -rf /var/lib/mysql.bak
```
以上补充的就是将数据盘挂载到MySQL数据目录的详细步骤。