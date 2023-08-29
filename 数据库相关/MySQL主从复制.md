MySQL主从复制是异步复制过程，底层是基于MySQL数据库自带的**二进制日志**功能。就是一台或多台MySQL数据库（slave，**从库**）从另一台MySQL数据库（master，**主库**）进行日志的复制然后再解析日志并应用到自身，最终实现**从库**的数据和主库保持一致。**MySQL主从复制是MySQL数据库自带的功能**，无需借助第三方工具。

MySQL复制过程：

* master将改变记录到二进制日志（binary log）
* slave将master的binary log拷贝到他的中继日志（relay log）
* slave重做中继日志中的事件，将其改变应用到自己的数据库中



**MySQL主从复制实现**

**配置主库**

1. 修改MySQL数据库配置文件/etc/my.cnf

   log-bin=mysql-bin # 启用二进制日志

   server-id=100 # 服务器唯一id

2. 重启MySQL服务

   systemctl restart mysqld

3. 登录数据库执行下面SQL

   ```sql
   grant replication slave on *.* to 'ss'@'%' identified by 'pwd';
   ```

   创建一个用户ss，密码为pwd，并且给ss用户授予replication slave权限。常用于建立复制时所需要的用户权限，也就是slave必须被master授权具备该权限的用户，才能通过该用户复制

4. 登录MySQL数据库，执行下面SQL，记录结果中**File**和**Postion**的值

   ```sql
   show master status;
   ```

   **注：执行上面SQL后不要执行任何操作（否则会改变position的位置）**

**配置从库**

1. 修改MySQL数据库的配置文件/etc/my.cnf

   ```sql
   server-id=191 # 服务器唯一id
   ```

   

2. 重启MySQL服务

   systemctl restart mysqld

3. 登录MySQL数据库，执行下面SQL

   ```sql
   change master to
   master_host='192.168.138.100',master_user='ss',master_password='pwd',
   master_log='上面记录的file名称',master_log_pos='上面记录的position';
   
   start slave;
   ```

4. show slave status; 查看从库状态