### 如何查询文件列表
```bash
# 查看文件的大小
ls -lh

# 查看文件列表显示时间格式化
ls -l --time-style '+%Y-%m-%d %H:%M:%S'
```

### 设置启动方式
```bash
# 查看当前的启动方式
systemctl get-default

# 设置为图形界面启动方式
systemctl set-default graphical.target

# 设置为文本模式启动方式
systemctl set-default multi-user.target
```

### 查看当前Linux系统信息
```bash
# 显示操作系统的名称、内核版本等信息。
uname -a

# 查看发行版信息。
cat /etc/redhat-rease

# 显示主机名。
hostname

# 查看内存使用情况。
free -h

# 显示磁盘空间使用情况
df -h

# 实时监控系统进程
top

# 查看进程信息
ps aux

# 获取 CPU 信息
cat /proc/cpuinfo

# 显示 CPU 的详细信息
lscpu

# 查看系统硬件信息
dmidecode
```

### 加密的方式在本地主机和远程主机之间复制文件（scp）
`scp` 命令 用于在 Linux 下进行远程拷贝文件的命令，和它类似的命令有 cp，不过 cp 只是在本机进行拷贝不能跨服务器，而且 scp **传输是加密的**。可能会稍微影响一下速度。当你服务器硬盘变为只读 read only system 时，用 scp 可以帮你把文件移出来。另外，scp 还非常不占资源，不会提高多少系统负荷，在这一点上，rsync 就远远不及它了。虽然 rsync 比 scp 会快一点，但当小文件众多的情况下，rsync 会导致硬盘 I/O 非常高，而 scp 基本不影响系统正常使用。
语法：
```bash
scp [选项] [参数]

# 选项列表
# -1：使用ssh协议版本1；
# -2：使用ssh协议版本2；
# -4：使用ipv4；
# -6：使用ipv6；
# -B：以批处理模式运行；
# -C：使用压缩；
# -F：指定ssh配置文件；
# -i：identity_file 从指定文件中读取传输时使用的密钥文件（例如亚马逊云pem），此参数直接传递给ssh；
# -l：指定宽带限制；
# -o：指定使用的ssh选项；
# -P：指定远程主机的端口号；
# -p：保留文件的最后修改时间，最后访问时间和权限模式；
# -q：不显示复制进度；
# -r：以递归方式复制。

# 参数
# 源文件：指定要复制的源文件。
# 目标文件：目标文件。格式为 user@host:filename（文件名为目标文件的名称）。

# 实例
# 01 从远程主机复制到本地
scp user@host:/data/demo.txt /data/
# 注意如果包含文件夹需要使用 -r 选项

# 02 上传本地文件到远程（与下载方式相同，只是目标换一个位子）
scp /data/demo.txt user@host:/data/
```