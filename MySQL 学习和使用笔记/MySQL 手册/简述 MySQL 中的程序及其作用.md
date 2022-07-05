- mysqld  
mysqld 是MySQL的一个守护进程，也就是MySQL程序的服务端，使用客户端可以与之进行链接。

- mysqld_safe  
mysqld_safe 是用了启动 mysqld 的一个脚本

- mysql.server  
mysql.server 也是服务程序启动脚本，它通过调用 mysqld_safe 来启动。

- mysqld_multi  
mysqld_multi 管理多个MySQL服务程序。