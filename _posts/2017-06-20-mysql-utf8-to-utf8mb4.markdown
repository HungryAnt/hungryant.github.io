---
layout: post
title:  "Mysql升级utf8mb4字符集 支持emoji字符"
date:   2017-06-20 14:01:30 +0800
categories: mysql
---

emoji表情的utf8编码占用4字节，而5.5以下版本的mysql支持的utf8编码，仅支持使用3字节，遇到4字节字符插入操作会直接报错

解决方案为升级mysql至5.5以上版本，配置mysql使用utf8mb4编码，历史数据进行编码转换

亲测 Windows Linux 阿里云RDS 均可支持，服务端程序无需做代码调整

具体操作步骤如下

## 升级mysql及更新配置

将mysql版本升级到5.5以上，以支持utf8mb4编码

mariadb是mysql的开源版本，推荐使用

- windows安装mariadb并支持utf8mb4

    - 下载地址： [https://downloads.mariadb.org/](https://downloads.mariadb.org/){:target="_blank"}

    - 修改配置

        修改`my.ini` （参考路径： C:\Program Files\MariaDB 10.2\data\my.ini）

        向`[mysqld]`中添加一行：
            
            character-set-server=utf8mb4

        最终内容如下：

            [mysqld]
            datadir=C:/Program Files/MariaDB 10.2/data
            port=3306
            innodb_buffer_pool_size=1014M
            character-set-server=utf8mb4
            [client]
            port=3306
            plugin-dir=C:/Program Files/MariaDB 10.2/lib/plugin


        重启mysql服务（右键我的电脑 -> 管理 -> 服务）

    - 查看字符集

        使用mysql命令行 查看当前字符集

            mysql> SHOW VARIABLES LIKE 'character_set_%';
            +--------------------------+-----------------------------------------------+
            | Variable_name            | Value                                         |
            +--------------------------+-----------------------------------------------+
            | character_set_client     | utf8                                          |
            | character_set_connection | utf8                                          |
            | character_set_database   | utf8mb4                                       |
            | character_set_filesystem | binary                                        |
            | character_set_results    | utf8                                          |
            | character_set_server     | utf8mb4                                       |
            | character_set_system     | utf8                                          |
            | character_sets_dir       | C:\Program Files\MariaDB 10.2\share\charsets\ |
            +--------------------------+-----------------------------------------------+

        看到`character_set_server`为`utf8mb4`，说明修改成功

- centos安装mariadb并支持utf8mb4

    - 安装方式

            yum install MariaDB-server MariaDB-client

    - 修改配置

        配置路径为`/etc/my.cnf`

        向[mysql]中添加`character-set-server=utf8mb4`，修改后内容如下

            [mysqld]
            datadir=/var/lib/mysql
            socket=/var/lib/mysql/mysql.sock
            # Disabling symbolic-links is recommended to prevent assorted security risks
            symbolic-links=0
            # Settings user and group are ignored when systemd is used.
            # If you need to run mysqld under a different user or group,
            # customize your systemd unit file for mariadb according to the
            # instructions in http://fedoraproject.org/wiki/Systemd

            character-set-server=utf8mb4

            [mysqld_safe]
            log-error=/var/log/mariadb/mariadb.log
            pid-file=/var/run/mariadb/mariadb.pid

            #
            # include all files from the config directory
            #
            !includedir /etc/my.cnf.d

    - 重启服务

            systemctl restart mariadb

        查看当前字符集
        
        ```sql
        mariadb root@localhost:(none)> SHOW VARIABLES LIKE 'character_set_%';
        Reconnecting...
        +--------------------------+----------------------------+
        | Variable_name            | Value                      |
        |--------------------------+----------------------------|
        | character_set_client     | utf8                       |
        | character_set_connection | utf8                       |
        | character_set_database   | utf8mb4                    |
        | character_set_filesystem | binary                     |
        | character_set_results    | utf8                       |
        | character_set_server     | utf8mb4                    |
        | character_set_system     | utf8                       |
        | character_sets_dir       | /usr/share/mysql/charsets/ |
        +--------------------------+----------------------------+
        ```
        
        看到`character_set_server`为`utf8mb4`，说明修改成功

- 阿里云rds配置支持utf8mb4

    - 配置mysql参数

        进入实例页面，点击左侧导航栏进入“参数设置”页面，参数名：`character_set_server`，运行参数值默认为`utf8`，将其修改为`utf8mb4`，点击右上方提交参数，实例稍后将重启支持新的编码

## 现有数据升级支持utf8mb4编码

对于现有的数据库及所有的数据表，进行编码转换，执行如下sql指令

数据库编码转换

```sql
ALTER DATABASE database_name CHARACTER SET = utf8mb4 COLLATE = utf8mb4_general_ci;
```

数据表编码转换

```sql
ALTER TABLE table_name CONVERT TO CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci;
```

对于新数据表，在创建时指定使用utf8mb4编码，例如：

```sql
CREATE TABLE `users` (
`id` BIGINT(20) NOT NULL AUTO_INCREMENT COMMENT '自增id',
`nickname` VARCHAR(32) NOT NULL COMMENT '昵称',
PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='用户';
```

使用阿里云rds注意下，经过编码转换后，使用`SHOW CREATE TABLE table_name`查看建表语句，部分表仍旧会展示`DEFAULT CHARSET=utf8`，观察发现这些表有个共同点，本省没有字符串列，猜测与RDS实现有关，可忽略