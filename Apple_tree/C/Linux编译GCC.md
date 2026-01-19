### 源码获取
- 官方站点：[https://ftp.gnu.org/gnu/gcc/](https://ftp.gnu.org/gnu/gcc/)
- 镜像站点：[https://gcc.gnu.org/mirrors.html](https://gcc.gnu.org/mirrors.html)

### 安装相关依赖

``` bash
# 安装EPEL源
yum -y install epel-release

# 安装工具
yum -y install wget vim gcc gcc-c++ make autoconf automake zip bzip2

# 安装基础编译软件
yum -y install gcc-gnat libgcc libgcc.i686 glibc-devel bison flex m4 texinfo build-essential
```

### 编译GCC

>GCC依赖gmp、mpfr、mpc，需先按序编译这三个依赖包

#### 编译gmp

``` bash
pushd gmp-VERSION/
./configure -C --prefix=/usr/local/gmp-VERSION
make && make install
popd
```

#### 编译mpfr

``` bash
pushd mpfr-VERSION/
./configure -C --prefix=/usr/local/mpfr-VERSION --with-gmp=/usr/local/gmp-VERSION
make && make install
popd
```

#### 编译mpc

``` bash
pushd mpc-VERSION/
./configure -C --prefix=/usr/local/mpc-VERSION --with-gmp=/usr/local/gmp-VERSION --with-mpfr=/usr/local/mpfr-VERSION
make && make install
popd
```

#### 编译GCC

``` bash
pushd mpc-VERSION/
mkdir objdir
pushd objdir
../configure -C --prefix=/usr/local/gcc-VERSION --with-gmp=/usr/local/gmp-VERSION --with-mpfr=/usr/local/mpfr-VERSION --with-mpc=/usr/local/mpc-VERSION
make && make install
popd
```

### 加载GCC环境

``` bash
export CC=/usr/local/gcc/bin/gcc
export CXX=/usr/local/gcc/bin/g++
export LD_LIBRARY_PATH="/usr/local/gcc/lib64:$LD_LIBRARY_PATH"
export PATH="/usr/local/gcc/bin:$PATH"
```