# MySQL数据备份

在任意路径下创建 `mysql_backup.sh` 文件（例如 `/data/mysql_backup.sh`），然后写入下列内容:

```bash
#!/bin/bash
#保存备份个数，备份31天数据
number=31
#备份保存路径
backup_dir=/root/backup
#日期
dd=`date +%Y-%m-%d-%H-%M-%S`
#备份工具
tool=mysqldump
#用户名
username=root
#密码
password=your_password
#将要备份的数据库
database_name=meta_data

#如果文件夹不存在则创建
if [ ! -d $backup_dir ];
then     
    mkdir -p $backup_dir;
fi

#简单写法 mysqldump -u root -p123456 users > /root/mysqlbackup/users-$filename.sql
$tool -u $username -p$password $database_name > $backup_dir/$database_name-$dd.sql

#写创建备份日志
echo "create $backup_dir/$database_name-$dd.dupm" >> $backup_dir/backup_log.txt

#找出需要删除的备份
delfile=`ls -l -crt $backup_dir/*.sql | awk '{print $9 }' | head -1`

#判断现在的备份数量是否大于$number
count=`ls -l -crt $backup_dir/*.sql | awk '{print $9 }' | wc -l`
if [ $count -gt $number ]
then
          #删除最早生成的备份，只保留number数量的备份
            rm $delfile
              #写删除文件日志
                echo "delete $delfile" >> $backup_dir/backup_log.txt
fi
```

为bash脚本开启权限:

```bash
chmod +x /data/mysql_backup.sh  # 开启权限；
```

创建定义定时任务:

```txt
(hot_topic) root@iZ2ze50qtwycx9cbbvesvxZ:/data# crontab -l
# 每天凌晨 0 点 0 分执行mysql数据备份，删除超过31天的备份。
0 0 * * * /data/mysql_backup.sh
```

