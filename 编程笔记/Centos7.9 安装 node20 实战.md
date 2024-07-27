## Centos7.9 安装 node20 实战
通常我们安装 nodejs 使用的是 `nvm` 工具，具体用法如下：
```bash
# 安装 nvm
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh | bash

# 安装指定的 node 版本
nvm install 22

# 查看是否安装成功
node -v
```

此时我收到报错：
```bash
bin/node: /lib64/libc.so.6: version `GLIBC_2.25' not found
```
根据翻阅资料，让我升级 `GLIBC` 版本：
```bash
wget http://ftp.gnu.org/gnu/glibc/glibc-2.28.tar.gz
tar xf glibc-2.28.tar.gz 
cd glibc-2.28/ && mkdir build  && cd build
../configure --prefix=/usr --disable-profile --enable-add-ons --with-headers=/usr/include --with-binutils=/usr/bin
```
这里会遇到各种问题：make、gcc、bison 等版太低

```bash
# 安装 gcc
yum install -y centos-release-scl
yum install devtoolset-8-gcc*
scl enable devtoolset-8 bash
gcc --version

# 安装 make
wget http://ftp.gnu.org/gnu/make/make-4.3.tar.gz
tar -xzvf make-4.3.tar.gz && cd make-4.3/
./configure  --prefix=/usr/local/make
make && make install
cd /usr/bin/ && mv make make.bak
ln -sv /usr/local/make/bin/make /usr/bin/make

# 安装 bison
yum install -y bison
```
注：`CentOS-SCLo-rh.repo`
```bash
# CentOS-SCLo-rh.repo
#
# Please see http://wiki.centos.org/SpecialInterestGroup/SCLo for more
# information

[centos-sclo-rh]
name=CentOS-7 - SCLo rh
baseurl=http://mirrors.aliyun.com/centos/7/sclo/$basearch/rh/
gpgcheck=1
enabled=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-SIG-SCLo

[centos-sclo-rh-testing]
name=CentOS-7 - SCLo rh Testing
baseurl=http://mirrors.aliyun.com/centos/7/sclo/$basearch/rh/
gpgcheck=0
enabled=0
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-SIG-SCLo

[centos-sclo-rh-source]
name=CentOS-7 - SCLo rh Sources
baseurl=http://vault.centos.org/centos/7/sclo/Source/rh/
gpgcheck=1
enabled=0
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-SIG-SCLo

[centos-sclo-rh-debuginfo]
name=CentOS-7 - SCLo rh Debuginfo
baseurl=http://debuginfo.centos.org/centos/7/sclo/$basearch/
gpgcheck=1
enabled=0
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-SIG-SCLo
```

最后在记录更新下 `GLIBC`
```bash
cd /root/glibc-2.28/build
../configure --prefix=/usr --disable-profile --enable-add-ons --with-headers=/usr/include --with-binutils=/usr/bin
make && make install 2>&1 &
tailf nohup.out
```
等待升级完成后，将新的动态链接库替换以前老的使之生效：
```bash
# 查看当前版本
strings /usr/lib64/libstdc++.so.6 | grep GLIBC

# 查看所有已存在的动态库
find / -name "libstdc++.so*"
# 将自己需要的新版本 gcc 动态库拷贝到该目录下
cp /home/gcc-5.2.0/gcc-temp/stage1-x86_64-unknown-linux-gnu/libstdc++-v3/src/.libs/libstdc++.so.6.0.21 /usr/lib64
# 移除老的
rm -rf libstdc++.so.6
ln -s libstdc++.so.6.0.21 libstdc++.so.6
# 然后再次查看
strings /usr/lib64/libstdc++.so.6 | grep GLIBC
```

