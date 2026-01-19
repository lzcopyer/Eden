- [仙人指路](https://docs.python.org/zh-cn/3.13/using/unix.html)

### Linux系统

#### 安装 IDLE[](https://docs.python.org/zh-cn/3.13/using/unix.html#installing-idle "Link to this heading")

在某些情况下，IDLE 可能未被包括在你的 Python 安装版中。

- 对于 Debian 和 Ubuntu 用户:
    
``` bash
sudo apt update
sudo apt install idle
```
    
- 对于 Fedora, RHEL 和 CentOS 用户:
    
``` bash
sudo dnf install python3-idle
```
    
- 对于 SUSE 和 OpenSUSE 用户:
    
``` bash
sudo zypper install python3-idle
```
    
- 对于 Alpine Linux 用户:
    
``` bash
sudo apk add python3-idle
```
    

### 在FreeBSD和OpenBSD上[](https://docs.python.org/zh-cn/3.13/using/unix.html#on-freebsd-and-openbsd "Link to this heading")

- FreeBSD用户，使用以下命令添加包:
    
``` bsd
pkg install python3
```
    
- OpenBSD用户，使用以下命令添加包:
    
``` bsd
pkg_add -r python
    
pkg_add ftp://ftp.openbsd.org/pub/OpenBSD/4.2/packages/<insert your architecture here>/python-<version>.tgz
```
    
- 例如：i386用户获取Python 2.5.1的可用版本:
    
``` bsd
pkg_add ftp://ftp.openbsd.org/pub/OpenBSD/4.2/packages/i386/python-2.5.1p2.tgz
```

### 自定义 OpenSSL[¶](https://docs.python.org/zh-cn/3.13/using/unix.html#custom-openssl "Link to this heading")

1. 要使用发行商的 OpenSSL 配置和系统信任存储库，请找到包含 `openssl.cnf` 文件或符号链接的目录，它位于 `/etc` 中。 在大多数发行版上该文件是在 `/etc/ssl` 或者 `/etc/pki/tls` 中。 该目录还应当包含一个 `cert.pem` 文件和/或一个 `certs` 目录。
``` bash
find /etc/ -name openssl.cnf -printf "%h\n"
/etc/ssl
```
    
2. 下载、编译并安装 OpenSSL。 请确保你使用 `install_sw` 而不是 `install`。 `install_sw` 的目标不会覆盖 `openssl.cnf`。
    
``` bash
curl -O https://www.openssl.org/source/openssl-VERSION.tar.gz
tar xzf openssl-VERSION
pushd openssl-VERSION
./config \
    --prefix=/usr/local/custom-openssl \
    --libdir=lib \
    --openssldir=/etc/ssl
make -j1 depend
make -j8
make install_sw
popd
```
    
3. 使用自定义的 OpenSSL 编译 Python （参考配置 `--with-openssl` 和 `--with-openssl-rpath` 选项）
    
``` bash
pushd python-3.x.x
./configure -C \
    --with-openssl=/usr/local/custom-openssl \
    --with-openssl-rpath=auto \
    --prefix=/usr/local/python-3.x.x
make -j8
make altinstall
```