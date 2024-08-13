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